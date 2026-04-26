---
name: tc-update
description: >-
  Update existing test cases in <tms> for <product>. Supports: bulk field
  updates (type, priority, status, automation), folder moves, content/steps
  edits, and renames. Handles etag flow automatically. ACTIVE TCs require
  explicit confirmation before any change.
  Trigger: "обнови кейс", "переименуй TC", "перемести в папку", "поменяй тип", "tc-update".
version: 0.1.0
---

# TC Update — <product> <tms>

Update existing test cases in `<tms>`. Handles etag automatically — the user never needs to think about it.

## Configuration

Replace `<tms>` with your test management system name and `mcp__<tms>__<tms>_*` with your actual MCP tool names. See `../tc-create/references/<tms>-api.md` for folder IDs.

## Hard Rules

- **ACTIVE TCs require explicit confirmation before any change.** Always stop, show what will change, and wait for approval. Only proceed after explicit yes in that message.
- **DRAFT and GUESS TCs** can be updated after the user confirms the plan (Step 4).
- Never change fields the user didn't ask to change.

---

## Supported Operations

| Operation | Example trigger |
|---|---|
| Bulk field update | "all TCs in the Auth folder → set type=SMOKE" |
| Folder move | "move TC-42 and TC-55 to Billing folder" |
| Steps / content edit | "update steps for TC-38" |
| Rename | "rename TC-71 to 'Payment Failed: provider=stripe'" |
| Mixed | "change priority and move to folder" |

---

## Workflow

### Step 1 — Clarify target and change

If not fully specified, ask:
- Which TCs? (explicit IDs, or filter: project + folder + status)
- What changes? (field=value, new folder, new title, new steps)

### Step 2 — Fetch target TCs

**By filter** (project + optional folder):
```
mcp__<tms>__<tms>_list_testcases(project_id=<id>)
```

**By explicit ID:**
```
mcp__<tms>__<tms>_get_testcase(id=<tc_id>)
```

Carry forward per TC: `id`, `status`, `title`, current field values. Etag is handled automatically by MCP.

### Step 3 — ACTIVE check

For each ACTIVE TC in the target set:
```
⚠️ TC-{id} "{title}" has status ACTIVE.
Planned change: {field} → {new_value}

Apply to this TC? (yes / skip)
```
Wait for answer per ACTIVE TC before continuing.

### Step 4 — Show plan, wait for confirmation

```
Updating N TCs:

  TC-{id} "{title}" [{status}]
    {field}: {old} → {new}

  TC-{id} "{title}" [{status}]
    ...

  ⏭ TC-{id} "{title}" — skipped (ACTIVE, user declined)

Confirm?
```

Only proceed after explicit yes.

### Step 5 — Execute updates

#### Field update, rename, steps edit

```
mcp__<tms>__<tms>_update_testcase(
  id=<tc_id>,
  title="...",         # rename
  priority=<1|2|3>,   # field update
  status="...",        # field update
  steps="step1\nstep2\n...",  # steps — plain text, one per line
)
```

Etag is fetched automatically — no manual etag management needed.

#### Folder move

```
mcp__<tms>__<tms>_add_to_folder(testcase_id=<id>, folder_id=<folder_id>)
```

Folder IDs: see `../tc-create/references/<tms>-api.md`.

### Step 6 — Report results

```
Done:
  ✅ TC-{id} "{title}" — updated
  ✅ ...
  ❌ TC-{id} — error: {http_error}
```

If any errors — show raw error response, don't silently skip.
