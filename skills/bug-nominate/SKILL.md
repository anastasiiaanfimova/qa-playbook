---
name: bug-nominate
description: >-
  Record a bug-dig verdict to the Bug Candidates <wiki> DB. Accepts the verdict
  from the current chat, checks for an existing row by fingerprint, then updates
  or creates. Does not investigate — that's bug-dig. Does not create <task-tracker> tasks —
  that's bug-create. Trigger: "bug-nominate", "запиши в кандидаты",
  "добавь в candidates", "зафиксируй вердикт", "номинируй баг".
---

# Bug Nominate — record verdict to Bug Candidates DB

One job: take a verdict from the current conversation and write it to the Bug Candidates <wiki> DB. No investigation, no <task-tracker> creation.

## Context

Bug Candidates DB is a discovery tool — it holds bugs surfaced by bug-review (weekly scan). Call bug-nominate when the investigated bug came from the Candidates DB or you explicitly want to track it there. Not every bug needs to be in <wiki> — bugs that go straight to <task-tracker> don't need a <wiki> row.

## Inputs

- Verdict already in the chat (from a bug-dig run)
- Or: explicit bug summary + verdict decision given by the user directly

## DB reference

| | Value |
|---|---|
| Data source ID | `26b9b9ff-6319-4e88-af44-b30a6978600d` |
| Data source URL | `collection://26b9b9ff-6319-4e88-af44-b30a6978600d` |
| Database page | https://www.<wiki>.so/c1e7bbdea35a4ccaa2aa115ef51d885e |

## Properties

| Property | Type | What to set |
|---|---|---|
| `Title` | title | Short symptom, 1 line |
| `Fingerprint` | rich_text | e.g. `<error-monitoring>:<your-error-monitoring-project>-3fqz`, `<task-tracker>:1214140404778710`, `manual:short-slug` — always lowercase |
| `Status` | select | `Active` (default), `Tracked` (<task-tracker> ticket exists), `Closed`, `Regression` |
| `Sources` | multi_select | `<error-monitoring>`, `<task-tracker>`, `<metrics>`, `<analytics>`, `<vcs>`, `<data-warehouse>`, `<wiki>` |
| `Signal refs` | rich_text | Raw pointers: <error-monitoring> IDs, <task-tracker> GIDs, commit hashes, <logs> queries |
| `First seen` | date | Today — only when creating a new row |
| `Last seen` | date | Today |
| `Weeks seen` | number | `1` for new rows; increment existing by 1 |
| `Severity hint` | select | `Critical` / `High` / `Medium` / `Low` |
| `<task-tracker> link` | url | URL if ticket already exists |
| `Trend` | select | `New` for new rows; leave unchanged for updates (bug-review recalculates on next weekly run) |
| `Verdict` | rich_text | Full verdict text from bug-dig or user summary |

Date format: `date:First seen:start = YYYY-MM-DD`, `date:First seen:is_datetime = 0`.

## Workflow

### Step 1 — Draft in chat

Show the user what will be written before touching <wiki>:
- Title, Fingerprint, Status, Severity hint, Verdict (truncated to ~3 lines)

Get confirmation ("ок", "пиши", "да") before proceeding.

### Step 2 — Check for existing row

```
mcp__notion__notion-search(
  query="<fingerprint>",
  filters={},
  data_source_url="collection://26b9b9ff-6319-4e88-af44-b30a6978600d"
)
```

`<wiki>-search` does **semantic search**, not exact match — it can return false positives for similar fingerprints. After getting results, fetch each candidate with `<wiki>-fetch` and verify that the `Fingerprint` property **contains** the searched fingerprint before updating. If no verified match → Step 4 (create).

### Step 3 — Update existing row

`mcp__notion__notion-update-page(page_id=<found_id>, command="update_properties", properties={...}, content_updates=[])`

Update: `Verdict`, `Status`, `Last seen`, `Severity hint`, `<task-tracker> link` (if now known).
Increment `Weeks seen` by 1.
Do NOT overwrite `First seen`.

### Step 4 — Create new row

`mcp__notion__notion-create-pages(parent={type:"data_source_id", data_source_id:"26b9b9ff-6319-4e88-af44-b30a6978600d"}, pages=[{properties:{...}}])`

Fill all fields. Set `First seen` = `Last seen` = today, `Weeks seen` = 1.

## Hard rules

- ❌ Never call without a verdict already in the current conversation.
- ❌ Never auto-run — user must explicitly call bug-nominate.
- ❌ Never create <task-tracker> tasks — that's bug-create.
- ✅ Always draft in chat and get confirmation before writing to <wiki>.
