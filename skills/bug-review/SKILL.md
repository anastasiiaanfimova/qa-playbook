---
name: bug-review
description: >-
  Methodology for surfacing likely bugs from system-wide signal convergence.
  Reads multiple independent telemetry sources (errors, logs, analytics,
  data warehouse, tracker, code), merges by fingerprint, ranks by weight,
  and produces a prioritized candidate list. Tool-agnostic.
---

# bug-review

A bug rarely shows up in only one place. The same broken behavior
emits an error event, drops an analytics conversion, accumulates rows
in a failure-table query, and may have a coincident commit. Looking
at any one source in isolation under-reports.

This skill captures the methodology of **multi-source signal
convergence**: pull from independent telemetry, merge fingerprints,
rank, and surface the most likely bugs without manually triaging
every source separately.

## Two independent time windows — never conflate

The skill operates over two windows simultaneously, used for
different questions:

### Delta window — "what's new since the last run"

Window: `[last_run, today]`. Used for **discovery** of new signals:
new error issues, new tracker tickets, new commits. Variable length;
depends on how often the review runs.

### Trend window — "is it getting worse?"

Window: **always** `this_week = [today-7d, today]` versus
`prior_week = [today-14d, today-7d]`. Fixed length; **independent of
last_run**. Used for rate/ratio/volume comparisons:

- Failure rate per area in the data warehouse
- Error class rate or ratio in logs
- Analytics event volumes / funnel conversion deltas
- Error-monitor frequency (events / users) week-over-week

If `last_run = today - 2 days`, you still compare last 7 days to the
previous 7 — otherwise deltas compress and look like "everything
improved" when really the window shrunk.

**Ratio formula doesn't depend on window length, only on equal
windows on both sides of the comparison.**

## Three-layer flow

Three layers, each independently re-runnable:

| Layer | Purpose | Output |
|---|---|---|
| **Refresh** | Pull deltas from each source, update per-source state, emit signals | List of `{fingerprint, title, source, severity_hint, signal_refs}` |
| **Bugs** | Merge signals into the candidates database via the single-writer skill (see `bug-nominate`) | Updated candidates table |
| **Synthesis** | Regenerate the human-readable summary view | Top-N candidates by score |

Layers can be invoked separately. Refresh-only after a partial
outage. Bugs-only after re-importing manually-collected signals. All
three for a regular cadence run.

## Sources — what each is good for

The methodology assumes you have access to several independent
telemetry sources. Each contributes signals of a specific shape:

| Source | Produces candidates? | Signal shape |
|---|---|---|
| Error monitor | Yes | New issues, frequency spikes, regression of resolved issues |
| Logs | Yes | Error-class rate spikes, new patterns |
| Analytics | Yes | Error events, funnel conversion drops, missing/dropped events |
| Data warehouse | Yes | Failure rate per business dimension (provider × tool × etc.) |
| Tracker | Yes | New bug tickets, status changes |
| VCS | Yes | Reverts, hot-file commits, critical-path touches |
| Wiki / docs | No (context only) | Updated specs, postmortems, internal investigations |

Manual sources (user reports in chat, social media mentions, support
channels) are intentionally **not** automated — they need human
judgement before becoming candidates.

## Cross-source merging — the same bug from different angles

A real bug typically surfaces in 2-4 sources at once. Cross-source
merging makes them one candidate, not four.

Common merge patterns:

- **Tracker → Error monitor.** Tracker ticket text often references an
  error-issue ID. A regex pull merges the tracker fingerprint into
  the error candidate, marking status `Tracked` and attaching the
  ticket link.
- **VCS → Error monitor.** A revert or hot-file commit on the same
  area as an active error candidate is strong correlation. Attach
  the commit refs.
- **Logs → Error monitor.** Same handler + same error class in two
  sources is one bug, not two.

Merge in a second pass after individual sources have produced their
own candidates — otherwise you can't tell what's new vs what's
already represented elsewhere.

## Trend lifecycle for candidates

Each candidate carries a status and a trend marker:

- **New** — first surfaced this run
- **Active** — surfaced this run, no related tracker
- **Tracked** — surfaced this run, has a tracker ticket
- **Declining** — was Active/Tracked, didn't surface this run, but
  not yet known-fixed
- **Gone** — confirmed resolved (error monitor resolved, ticket
  closed)
- **Closed** — three Declining runs in a row → auto-archive
- **Regression** — fingerprint matches a Closed entry → reopen as
  Active

Don't delete Closed entries. They're the seed for regression
detection — a fingerprint reappearing after Closed is a stronger
signal than a brand-new fingerprint.

## Idempotency under flexible cadence

The default cadence is weekly, but daily runs are useful for actively
monitored periods (post-incident, near release) — provided counters
are idempotent within an ISO week.

Concretely: the "weeks seen" counter on a candidate increments
**only when the ISO week of the last-seen date changes**. Multiple
runs in the same ISO week don't inflate the count. This makes daily
or twice-daily runs safe.

## Never delete, only mark

Closing or archiving is reversible (trend = Closed). Deleting is
not. The candidate database is the institutional memory of "what was
broken once" — losing rows loses regression-detection value.

## Output shape

After a full run, summarize to chat:

```
Weekly review YYYY-MM-DD — done
  Sources refreshed: N/M
  Candidates: X new | Y regressed | Z auto-closed | T active total
  Top 5 by score:
    1. [High] <title> (<sources>, <weeks seen>)
    ...
  Run deep investigation on top candidates.
```

Don't dump every row. The full table is in the database; chat is for
overview + the few things demanding attention now.

## Workflow per source (Layer 1: Refresh)

For each source:

1. **Find `last_run`.** Read the per-source state page; pull the
   stored last-run date. If absent, default to 7 days back and add
   the marker.
2. **Collect deltas in two windows.**
   - Discovery (new signals) → `[last_run, today]` window.
   - Trend signals (rates/ratios) → fixed `last 7d vs prior 7d`,
     independent of `last_run`.
3. **Append to source page** under `## Updates YYYY-MM-DD`. Bump
   `last_run`. Never rewrite old content — append-only preserves the
   audit trail.
4. **Emit signals** for Layer 2 to process.

## Workflow for candidates (Layer 2: Bugs)

Delegate writes to the single-writer skill (`bug-nominate`) in
silent mode — do not write directly. Layer 2's responsibilities:

1. Build a `fingerprint → existing-row` map from the database
2. For each incoming signal, decide create / update / regression by
   the existing fingerprint match
3. Determine candidate severity and rank score
4. Hand args to the writer, including a minimal body for new
   candidates (Symptom + Signal refs only — full investigation is
   added later by deep-dive)
5. Mark non-matched Active/Tracked candidates as Declining or Gone
   based on whether their source signal still exists
6. Log the run summary to chat

Layer 2 doesn't perform deep investigation — that's a separate skill.
It's a fast pattern-match across many signals; investigation is slow
craft on a few of them.

## Workflow for synthesis (Layer 3)

Regenerate the human-readable QA review document from scratch. Pull
top-N candidates by score, summarize per-source updates, compute
week-over-week deltas. Stable section structure makes WoW diffs
readable.

## Hard rules

- ✅ Append-only updates to source pages with date headers
- ✅ Idempotent counters keyed on ISO week
- ✅ Cross-source merge as a second pass after per-source collection
- ✅ Delegate writes to a single-writer skill — don't fan out
- ❌ Never delete candidates — mark Closed instead
- ❌ Never use the delta window for trend rate comparisons
- ❌ Never auto-create tracker tickets from review output
- ❌ Never include manual sources (chat reports, support, social) in
  the automated candidate list — they require human triage first

## Anti-patterns

- ❌ Single-source review — under-reports systematically
- ❌ Comparing rates over windows of different lengths — false
  improvements appear
- ❌ Non-deterministic fingerprints (timestamps, free text) — every
  run looks like "all new"
- ❌ Bumping counters per run instead of per-ISO-week — counts
  inflate, signal degrades
- ❌ Deleting Closed candidates — regression detection breaks
- ❌ Mixing investigation prose into review output — Layer 2 is fast
  triage, deep-dive is a separate skill
- ❌ Letting per-source state drift (no `last_run` marker, no
  append-only updates) — runs become non-idempotent
