---
name: tc-gap
description: >-
  Methodology for finding what your test suite doesn't cover. Cross-references
  existing test cases against three independent signal sources — product
  analytics, code, and admin operations — and surfaces uncovered areas as
  prioritized skeleton TCs. Tool-agnostic.
---

# tc-gap

A test suite is a claim about coverage. Gap analysis verifies the claim by
asking: for everything we know happens in the product, is there a TC?
"Everything we know happens" comes from three independent sources, not
gut feel.

## Three signal sources, one per layer

| Layer | Primary signal | What it answers |
|---|---|---|
| **Web / client** | Product-analytics events with their property values | What users actually do |
| **Backend** | Handler functions (HTTP routes, queue consumers) discoverable in code | What the system can do |
| **Admin / internal** | Pages and operations in the admin app | What the team can do |

Each source is independent — events tell you what users emit, handlers
tell you what the server accepts, admin pages tell you what operators
can change. A bug in any layer can hide for a long time if you only
look at one signal source.

Optional fourth source: **error-monitoring**. Unresolved exceptions
without matching TCs are gaps you didn't know existed. Use this as
enrichment, not as a primary axis — error logs reveal what's failing,
not what's untested.

## Output: skeletons, not full TCs

Gap analysis produces **skeleton test cases** with metadata pointing at
the source signal — not finished TCs ready to run. The skeleton carries:

- `Source` — which signal type (analytics event / handler / admin page)
- `SourceRef` — the specific event name, file path, or page name
- `GapReason` — why we think this needs a TC (no match found / new since
  last analysis / signal changed)
- `DetectedAt` — date

A separate `tc-create` pass picks skeletons from the queue, does the
research, fills in real steps, and bumps status from `BACKLOG` to
`GUESS` or `DRAFT`. This split matters: gap analysis is a fast pattern
match across hundreds of signals; writing real steps is slow craft.
Don't conflate them.

## Cross-reference algorithm

For each signal in each source, ask "is there an existing TC whose title
matches?". Match definitions:

- **Web (event-based):** TC title contains all significant words from
  the event name. Case-insensitive. `Task Created` matches a TC titled
  "Task Created: type=video"; `Task Started` does not.
- **Backend (area-based):** TC title contains at least one keyword from
  the handler area description. Looser by design — backend areas are
  broad ("billing webhooks") and many TCs map to one area.
- **Admin (page+operation):** TC title contains the page or operation
  keyword.

Word matching is fuzzy. False positives happen ("Post Scheduled" can
falsely match "Post Published: publishType=scheduled"). Better to err
toward flagging the gap and let `tc-create` discard than to miss it.
When unclear, mark as gap with an ambiguity note.

## Priority heuristic for skeletons

Without doing real research, classify by signal weight:

| Priority | Signal characteristics |
|---|---|
| HIGH (1) | Critical user path (auth / payment / core feature / data integrity); error-monitoring shows >1000 events/day |
| MEDIUM (2) | Mainstream feature without coverage; error-monitoring 100-1000/day |
| LOW (3) | Edge case; view-only; <100/day |

The downstream `tc-create` pass will refine priority with real research.
Skeletons just need a coarse sort order so important gaps surface first.

## Deduplication before creating skeletons

Before writing a skeleton, check the project for existing TCs in any
"unfinished" status (`BACKLOG`, `GUESS`, `DRAFT`) whose title overlaps
≥2 significant words with the proposed title. If found, skip — don't
duplicate. This keeps the queue clean across repeat runs.

## STALE marking — coverage drift in the other direction

Existing TCs can lose their signal:

- Handler renamed or removed → backend TC points at nothing
- Analytics event dropped from taxonomy → web TC has no source
- Admin page removed → admin TC points at nothing
- Error-monitoring issue resolved and gone → TC for that error is no
  longer about a real failure

Mark these `STALE` rather than deleting. They carry history, and the
underlying behavior may resurface. Append the new reason to `GapReason`
so the trail is preserved.

If a TC is *already* `STALE` and the same signal-loss recurs, just
update the reason note. Don't create a second STALE marker.

## When to run

Once a week is a useful default — frequent enough to catch new
features, rare enough that handler maps stabilize between runs. Run
after a code refresh (so the handler map reflects what's actually
deployed) and against current analytics taxonomy (not yesterday's
cached event list).

## Fallback when a signal source is unavailable

- **Analytics down** → skip web cross-reference; check critical paths
  (payment / auth / core features) by hand.
- **Code unavailable / not synced** → use a hand-maintained handler
  map for the backend axis. Looser matching, but better than skipping.
- **TMS write disabled** → produce the report in chat only. Don't lose
  the analysis just because creation is blocked.

## Output format

```
## TC Gap — <date>
TMS: N total (Web: X | Back: Y | Admin: Z)

### New BACKLOG skeletons created: N
[list: TC title | project | priority | source]

### Marked STALE: N
[list: TC id + title | reason]

### Coverage unchanged: N areas
```

Keep this in chat — it's a snapshot, not a document. The skeletons
themselves persist in the TMS as the durable artifact.

## Anti-patterns

- ❌ Running gap analysis against stale code clones — you'll miss new
  handlers and falsely flag renamed ones as STALE
- ❌ Creating full TCs from gap analysis — that's `tc-create`'s job;
  conflating produces hasty, low-quality TCs
- ❌ Treating fuzzy false-positives as confirmed coverage — when in
  doubt, flag the gap, don't suppress it
- ❌ Re-running without dedup → BACKLOG fills with near-duplicates
- ❌ Hard-deleting STALE TCs — history matters for "was this ever
  tested?"
