---
name: bug-candidates
description: >-
  Weekly refresh of <product> QA data sources + rebuild of Bug Candidates list.
  Two-layer flow: refresh individual source pages in <wiki> (<vcs>, <error-monitoring>,
  <metrics>, <analytics>, <data-warehouse>, <task-tracker>, <wiki> docs), then rebuild the "Bug
  Candidates" <wiki> DB with smart dedup against prior week. Main output is
  Bug Candidates — the input for bug-dig triage. Never auto-creates <task-tracker>
  tasks. Trigger: "bug-candidates", "weekly-review", "обнови ревью", "собери кандидатов", "свежий bug list".
version: 0.2.0
---

# Bug Candidates — <product> QA refresh flow

Purpose: keep the QA Review knowledge base fresh without spamming <wiki> with append-only logs, and produce a ranked list of bug candidates for manual triage with `bug-dig`.

**The single load-bearing output is the Bug Candidates <wiki> DB.** Everything else (source-page updates, QA Review synthesis) is supporting context.

## Sub-commands

| Command | What it does |
|---|---|
| `/bug-candidates refresh [source?]` | Layer 1. Update source pages in <wiki>. Without arg — all auto sources. With arg (`<vcs>`, `<error-monitoring>`, `<task-tracker>`, `<metrics>`, `<analytics>`, `<data-warehouse>`, `<wiki>-docs`) — one. |
| `/bug-candidates bugs` | Layer 2. Rebuild Bug Candidates DB — smart merge against existing rows. |
| `/bug-candidates qa` | Regenerate QA Review synthesis on top of fresh sources + candidates. |
| `/bug-candidates all` | Everything in order: refresh → bugs → qa. |

**Default cadence: weekly.** Don't run more than once a week without explicit reason — noise in Last seen / Weeks seen columns.

## Sources — scope of this skill

Automated (in scope):
- **<error-monitoring>** — new unresolved issues since last run, top regressions
- **<vcs>** — commits across 3 repos (`backend`, `frontend`, `admin`) since last run
- **<task-tracker>** — BUG-type tasks created/modified since last run, open tasks by priority
- **<metrics> / <logs>** — prod error spikes vs baseline
- **<analytics>** — new event names, funnel drops
- **<data-warehouse>** — tool failure rates week-over-week
- **<wiki> docs** — pages with `last_edited_time` > last run

Manual (NOT in this skill — user handles separately):
- Trustpilot
- Telegram dev chat

## Configuration references

See `references/<wiki>-schema.md` for the Bug Candidates DB schema and IDs.
See `references/merge-algorithm.md` for the dedup/update logic.
See `references/fingerprints.md` for how fingerprints are computed per source.
See `references/sources/*.md` for per-source collectors (queries, severity mapping).

## Workflow

### Layer 1: Refresh (`refresh`)

For each source in scope (or the one requested):

1. **Find `last_run`.** Read the source's <wiki> page — look for `<!-- last_run: YYYY-MM-DD -->` HTML comment in the first block. If missing, use 7 days ago.
2. **Collect delta** using the recipe in `references/sources/<source>.md`. Limit window: `[last_run, today]`.
3. **Update the source page** — prepend a new section `## Updates YYYY-MM-DD` with the delta, bump the `last_run` marker. Never rewrite older sections.
4. **Emit signals** (in-memory) — a list of `{fingerprint, title, source, severity_hint, signal_refs}` entries that Layer 2 will consume.

Source pages to update:
- <vcs> → `https://www.<wiki>.so/34898d8a3c9b8170a985e4b16841e510`
- <error-monitoring> → NEW — create under Review as "<error-monitoring>" if missing
- <task-tracker> → `https://www.<wiki>.so/34898d8a3c9b81d0a060e2aa8665e537`
- <metrics> → `https://www.<wiki>.so/34898d8a3c9b81c0adeffc4de6a2f0ec`
- <data-warehouse> → `https://www.<wiki>.so/34898d8a3c9b818c9e54f814d43bc10f`
- <analytics> → `https://www.<wiki>.so/34b98d8a3c9b813da281dbd908b6195b`
- <wiki> docs → `https://www.<wiki>.so/34898d8a3c9b81b097efcd6b96bb1d4a`

**First-run caveat.** If this is the first `/bug-candidates` run, source pages don't have `last_run` markers yet. Use 7 days back and add the marker. <error-monitoring> page may not exist — create it under Review as a sibling of <vcs>.

### Layer 2: Bugs (`bugs`)

Apply the merge algorithm from `references/merge-algorithm.md` to the Bug Candidates DB.

High-level:
1. Query all existing rows from the Bug Candidates DB.
2. Build fingerprint → row_id map.
3. For each incoming signal from Layer 1 (or a fresh collection if run standalone):
   - `fingerprint in existing & status != Closed` → UPDATE: bump `last_seen`, increment `weeks_seen`, recompute `trend`.
   - `fingerprint in existing & status == Closed` → REGRESSION: set `status=Regression`, `last_seen=today`, reset `weeks_seen=1`, push back to top.
   - New fingerprint → CREATE: `status=Active`, `trend=New`, `first_seen=last_seen=today`, `weeks_seen=1`.
4. For each existing Active/Tracked row NOT matched this run:
   - Check the source — if <error-monitoring> issue resolved / <task-tracker> task closed → `status=Closed`, `trend=Gone`.
   - Else → leave status, set `trend=Declining`, DO NOT bump `last_seen`.
   - If `trend=Declining` for 3 weeks in a row → auto `status=Closed`.
5. Compute severity hint and rank score per row (see `references/merge-algorithm.md`).
6. Log summary: `N new | M regressed | K closed | total active = X`.

**Never delete rows.** Closed rows stay for regression detection and historical stats.

### Layer 3: QA synthesis (`qa`)

Regenerate the QA Review page (`https://www.<wiki>.so/34898d8a3c9b81ffb128c375efe0f113`) from scratch — this is the user-facing summary. Pull from:
- Bug Candidates (top Active rows by score)
- Source-page `## Updates` sections from the last run
- High-level deltas: "N new bugs this week", "M regressions", "K tracked → closed"

Rewrite the page content. Keep structure stable so week-over-week diff is legible in <wiki> history.

## Anti-patterns — check before finishing a run

- ❌ **Appending to source pages without a diff section header.** Always use `## Updates YYYY-MM-DD` as a clear marker so manual review can tell what came from automation.
- ❌ **Skipping the `last_run` marker update.** Next run will re-fetch the same window = duplicate signals.
- ❌ **Deleting Closed rows.** Archive is load-bearing for regression detection.
- ❌ **Auto-creating <task-tracker> tasks.** Never. Bug Candidates is triage-only; `bug-dig` is the next step, and tickets are a manual decision.
- ❌ **Computing fingerprints non-deterministically** (timestamps, free-text). Fingerprint must be reproducible across runs — see `references/fingerprints.md`.
- ❌ **Running twice in one day.** Weeks seen becomes meaningless. Check `last_seen` of any existing row — if ≥ 1 matches today, warn the user and bail unless they confirm.
- ❌ **Mixing manual Trustpilot / Telegram signals into Bug Candidates automatically.** Those are user-entered — if the user wants to add one, it's a separate manual row.

## Output

At the end of `all` or `bugs`, post a short summary in chat:

```
Weekly review 2026-04-24 — done
  Sources refreshed: 7/7
  Candidates: 4 new | 1 regressed | 3 auto-closed | 18 active total
  Top 5 by score:
    1. [High] Stripe webhook signature missing  (<error-monitoring> + <task-tracker>, 3 weeks)
    2. [High] Payblis webhook double credits    (<error-monitoring>, NEW)
    ...
  Run bug-dig on top candidates for verdicts.
```

Don't dump all rows. User sees full table in <wiki>.

## Source collectors — full list

All 7 automated collectors are implemented in `references/sources/`:

| Source | Produces candidates? | Recipe |
|---|---|---|
| <error-monitoring> | Yes | `sources/<error-monitoring>.md` |
| <vcs> | Yes (risk signals only — revert/hot-file/critical-path) | `sources/<vcs>.md` |
| <task-tracker> | Yes (open BUG tasks) | `sources/<task-tracker>.md` |
| <metrics> / <logs> | Yes (error-rate spikes) | `sources/<metrics>.md` |
| <analytics> | Yes (error events + funnel drops); new events = Low/FYI | `sources/<analytics>.md` |
| <data-warehouse> | Yes (tool failure rate) | `sources/<data-warehouse>.md` |
| <wiki> docs | **No** — context-only, source page update only | `sources/<wiki>-docs.md` |

**First-run caveats** per collector:
- <error-monitoring> source page under Review doesn't exist yet — create on first <error-monitoring> refresh.
- <data-warehouse> schema must be derived from a dashboard panel on first run (Option A in `<data-warehouse>.md`) and cached in that file.
- <wiki>-docs watched-doc ID list needs to be resolved and cached on first run.

**Cross-source merge on ingestion:**
- <task-tracker> → <error-monitoring>: regex `<ERROR-ID>-XXX` in <task-tracker> task notes merges the row into the <error-monitoring> candidate (adds `<task-tracker>:` fingerprint, sets Status=Tracked, fills <task-tracker> link).
- <vcs> → <error-monitoring>: hot-file classifier reads Active candidate signal refs and fingerprints commits against them.
- <metrics> → <error-monitoring>: if same handler + error class show up in both, the run merges fingerprints in a second pass (Layer 2 post-merge cross-check).
