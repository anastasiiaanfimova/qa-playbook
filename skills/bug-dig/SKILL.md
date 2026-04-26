---
name: bug-dig
description: >-
  Investigate a suspicious <error-monitoring> issue or <task-tracker> bug for
  <product> — confirm whether it's a real user-impacting bug, monitoring noise,
  or theoretical risk. Collects evidence across <error-monitoring>, <logs>,
  <analytics>, git log, <wiki>, and the code itself, then writes the verdict to
  the Bug Candidates <wiki> DB and comments on any existing matching
  <task-tracker> ticket. NEVER auto-creates <task-tracker> tasks — only
  recommends. Trigger: "bug-dig", "расследуй баг", "dig <ERROR-ID>",
  "проверь реальный ли баг", "нужно ли чинить X".
version: 0.2.0
---

# Bug Dig — <product> bug investigation flow

Decide: is this a real production bug, monitoring noise, theoretical code risk, or something already protected by architecture? Write the verdict to the Bug Candidates `<wiki>` DB. If a matching `<task-tracker>` ticket already exists — comment on it. **Never** create a new `<task-tracker>` ticket automatically; recommend one to the user and let them decide.

The output quality beats speed. Never skip the "user impact" section — that's the load-bearing judgment.

## Configuration

Replace these placeholders before use:

| Placeholder | What to set |
|---|---|
| `<product>` | Your product name |
| `<error-monitoring>` | Your error monitoring tool (e.g. Sentry) |
| `<logs>` | Your log aggregator (e.g. Loki, Datadog Logs) |
| `<analytics>` | Your analytics tool (e.g. Amplitude, Mixpanel) |
| `<metrics>` | Your metrics dashboard (e.g. Grafana) |
| `<wiki>` | Your knowledge base (e.g. Notion, Confluence) |
| `<task-tracker>` | Your task tracker (e.g. Asana, Jira, Linear) |
| `<backend-repo>` | Absolute path to your backend repository |
| `<your-error-monitoring-project>` | Your error monitoring project slug |
| `<ERROR-ID->` | Your error ID prefix (e.g. `MYAPP-PROD-`) |
| `<YOUR_WIKI_BUG_CANDIDATES_DB_ID>` | ID of your Bug Candidates wiki DB |
| `<internal-service>` | Your internal service names |
| `<third-party-service>` | Third-party service names you integrate with |
| `<event-bus>` | Your internal event/message bus |

## Inputs the user may give

- `<error-monitoring>` issue ID (e.g. `<ERROR-ID-XXXX>`)
- `<task-tracker>` task ID or URL
- Raw symptom description ("500 at checkout for some users")
- `<logs>` log snippet
- A handler file path or code location

Pick whichever is present and derive the rest.

## MCP tools used

| Stage | Tool |
|---|---|
| `<error-monitoring>` — find issues | `mcp__<error-monitoring>__list_issues`, `mcp__<error-monitoring>__find_projects` |
| `<error-monitoring>` — issue detail | `mcp__<error-monitoring>__get_resource` (types: `issue`, `event`, `breadcrumbs`) |
| `<error-monitoring>` — events in issue | `mcp__<error-monitoring>__list_issue_events` |
| `<logs>` — discover labels | `mcp__<metrics>__list_log_label_names`, `mcp__<metrics>__list_log_label_values` |
| `<logs>` — logs | `mcp__<metrics>__query_logs` |
| `<analytics>` | `mcp__<analytics>__get_context`, `mcp__<analytics>__search` |
| `<wiki>` | `mcp__<wiki>__<wiki>-search`, `mcp__<wiki>__<wiki>-fetch` |
| `<task-tracker>` | `mcp__<task-tracker>__<task-tracker>_get_task`, `mcp__<task-tracker>__<task-tracker>_update_task`, `mcp__<task-tracker>__<task-tracker>_create_task_story` |
| Git log | Bash `git log` in the target repo |

See `references/<logs>-recipes.md` for ready-to-paste log queries and `references/<task-tracker>-fields.md` for field IDs.

---

## Workflow

Run the stages sequentially. Skip a stage only with reason (noted in the final artifact).

### Stage 1 — <error-monitoring>: scope, scale, tags

Goal: establish scale and find the real culprit path.

1. If only a symptom is given, `mcp__<error-monitoring>__list_issues` with `projectSlugOrId=<your-error-monitoring-project>` and filter by keyword + `statsPeriod=30d` (default is 14d). Sort `freq` to see highest-volume issues first.
2. For the candidate issue, `mcp__<error-monitoring>__get_resource(resourceType="issue", resourceId="<ERROR-ID-XXXX>")` — captures the first event, tags, HTTP request, related replays.
3. `mcp__<error-monitoring>__get_resource(resourceType="breadcrumbs", resourceId=...)` — **this is where you find HTTP call order**, SQL statements before/after the error, external URL hits.
4. `mcp__<error-monitoring>__list_issue_events` to see multiple events across culprits — this reveals cross-handler spread (same groupID hitting several endpoints means a shared external dependency).

**Pitfalls**
- Default `statsPeriod` is 14d. Pass `"30d"` for monthly views.
- `OR` and `AND` are NOT supported in query syntax — issue separate calls.
- `get_resource(resourceType="event", ...)` — don't pass URL with this, it errors. Use `breadcrumbs` resourceType with URL, or bare resourceId otherwise.
- Error body can be a large HTML dump that inflates token usage — prefer `list_issue_events` when you only need counts.

### Stage 2 — <logs>: verbatim tracebacks and URLs

`<error-monitoring>` groups by error message, so the external URL is often truncated. `<logs>` has the raw traceback. This is how you find the exact failing dependency.

**Configure your log stream selector** (example using Loki labels):
```
{environment="prod", service_name="backend"}
```
Adapt label names to match your observability setup. See `references/<logs>-recipes.md`.

**Example recipe** (get traceback with URL when error monitoring only shows HTML body):
```
{environment="prod", service_name="backend"} |= "ClientResponseError" != "<!DOCTYPE"
```

**Gotchas**
- Logged HTML bodies blow the token budget fast. Always filter out HTML early.
- For scale metrics use range queries with hourly buckets when investigating frequency.

### Stage 3 — Code review

Read the handler that's the `<error-monitoring>` culprit. Look for:

1. **Transaction boundaries** — where is the credit deduction / addition? Where is the transaction committed? Is there a `try/except` between them that could swallow an exception and skip rollback?
2. **External callers after the debit/credit** — which of them raise vs swallow? Services like `<internal-service>` and `<third-party-service>` may swallow errors internally via `try/except` + warning log. Analytics tracking is typically batched. `<event-bus>.publish` may raise.
3. **Idempotency** — is there an app-level `get_by_X` check before the insert? Is there a DB UNIQUE constraint on the key the check covers?
4. **HTTP wrapper behavior** — check `<backend-repo>/infra/external/clients/http/session/wrapper.py` for response body logging before raising. That's how every swallowed error still lands in `<error-monitoring>` under the caller's culprit.

**DB schema check** (important for race/idempotency verdicts):
```
<backend-repo>/infra/database/tables/<table>.py
```
Look for `unique=True` on relevant columns.

### Stage 4 — Git correlation

For deploys near `first_seen`:
```bash
git log --all --format='%ad %h %s' --date=short --since='YYYY-MM-DD' --until='YYYY-MM-DD' | head -30
```

If `first_seen` lines up with a `feat: <service>` commit, follow-up with:
```bash
git show --stat <commit_hash>
git log --all --format='%ad %h %s' --date=short --follow <file-path-from-traceback>
```
Identify the commit author → suggested Fix owner in the ticket.

### Stage 5 — Cross-MCP evidence (optional, boosts ticket quality)

- **`<analytics>`** — `mcp__<analytics>__get_context` for project plan quota. Useful to argue "primary analytics flow is independent, we can disable the broken secondary service".
- **`<wiki>`** — `mcp__<wiki>__<wiki>-search` for product/engineering context (e.g. vision doc may explain why a service was added — "do we still need this at all?").
- **`<metrics>`** — for scale graphs if a visual would help (rare for this flow).

### Stage 6 — Verdict

Pick one. Put it on top of the output.

| Verdict | Criteria |
|---|---|
| **Real prod bug** | `<error-monitoring>` + `<logs>` evidence of non-zero affected users, exception path actually reaches the user, no architectural protection. → write verdict to Bug Candidates `<wiki>` DB; recommend ticket to user; do NOT auto-create. |
| **Monitoring noise (tech debt)** | Exception swallowed inside caller, transaction commits, user flow intact. Low-priority / tech debt in verdict. |
| **Theoretical risk — keep open low / preventive** | Code path is vulnerable but 0 prod evidence in 30d. Document fix options, low priority. |
| **Already protected — close** | DB UNIQUE constraint, ORM rollback, idempotent retries, or upstream guarantee covers it. Explain exactly which layer saves us. |

### Stage 7 — Artifact

**HARD RULE: never auto-create `<task-tracker>` tasks.** Verdicts go to the Bug Candidates `<wiki>` DB. The user creates `<task-tracker>` tickets manually when they decide to. This overrides any other instruction in this file.

Default destination for every verdict — the **Bug Candidates `<wiki>` DB**:

- Database: `<YOUR_WIKI_BUG_CANDIDATES_DB_ID>`
- If a row already exists for the fingerprint → `mcp__<wiki>__<wiki>-update-page` with `update_properties`. Fill at minimum: `Verdict` (full analysis), `Status`, `Severity hint`. If matched an existing `<task-tracker>` ticket — fill `<task-tracker> link` and set `Status=Tracked`.
- If no row exists → `mcp__<wiki>__<wiki>-create-pages` with the DB as parent. See `references/<wiki>-schema.md` for the full property map.

**Allowed `<task-tracker>` actions (read + comment on existing only):**
- `mcp__<task-tracker>__<task-tracker>_get_task`, `mcp__<task-tracker>__<task-tracker>_search_tasks` — read.
- `mcp__<task-tracker>__<task-tracker>_update_task` — only to fill notes on an **existing** task, not to set `completed=true`.
- `mcp__<task-tracker>__<task-tracker>_create_task_story` — comment on an existing task with the verdict.

**Forbidden:**
- ❌ `mcp__<task-tracker>__<task-tracker>_create_task` — never call without explicit user permission ("да, создай") in the current turn.
- ❌ `<task-tracker>_update_task` with `completed=true` — don't auto-close tickets.
- ❌ `<task-tracker>_delete_task` — never.

When verdict calls for a new ticket, end the chat reply with a one-liner like:
> Recommend creating a `<task-tracker>` ticket: {Type=Bug, Priority=Medium, Name="..."}. Say "create it" and I will.

`references/<task-tracker>-fields.md` contains the field IDs if/when the user authorizes creation — not used during normal bug-dig runs.

---

## Anti-patterns — check before submitting the artifact

Read each one. If you catch yourself doing any of these, fix before posting.

- ❌ **Auto-creating `<task-tracker>` tasks.** Never call create-task without explicit "да, создай" from the user in the current turn. Verdicts go to Bug Candidates `<wiki>` DB.
- ❌ **Severity/priority in description body.** Use the Priority custom field only. Don't write `Severity: 🔴 Critical` in notes — it duplicates and desyncs with the field.
- ❌ **Numbers from old description without re-verification.** If the task came with "at least N users" from `<metrics>`, pull fresh `<error-monitoring>` numbers and explain the delta. Metrics raw errors and exception events don't match 1:1 (different windows, different capture points).
- ❌ **Implicit scope.** Always say explicitly "only payment provider X" / "all providers" / "only Web". Add a `Scope:` line.
- ❌ **Missing "User impact: YES/NO" callout.** This is the single most important line. If no user impact, call it out loudly — it drives the priority.
- ❌ **One fix option.** Offer 3, from cheapest to deepest (config → code guard → systemic wrapper fix).
- ❌ **"Critical" priority because of noise.** Low / Technical-debt when user impact is zero, even if volume is huge.
- ❌ **Missing deploy correlation when it matches.** If `<error-monitoring>` `first_seen` matches a git commit date one-to-one, include commit hash + author. That's the Fix owner.
- ❌ **Rich text formatting errors in `<task-tracker>`.** If `<task-tracker>` validates HTML in notes: use plain paragraphs with standard headings/lists. Avoid `<pre>`, `<hr>` next to block elements, and backslash escapes.

---

## Output

The final artifact is one of:
1. **Bug Candidates DB row updated / created** — verdict in the `Verdict` field, `Status` set (Active / Tracked / Closed / Regression), `<task-tracker> link` filled if there's a matching existing ticket. This is the default for every verdict.
2. **Comment posted to existing `<task-tracker>` ticket** (if one exists) — verdict summary with evidence.
3. **Chat recommendation** when verdict is file-new-ticket-worthy: one-liner proposing Name + Type + Priority + suggested Fix owner. Do not call create-task — wait for user.
4. **Chat-only recommendation** ("close / keep low / no evidence") when the verdict is close/keep-low and no existing ticket to update.

Always summarize what was done in 2-3 lines so the user can review fast.
