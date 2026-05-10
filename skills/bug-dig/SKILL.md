---
name: bug-dig
description: >-
  Methodology for investigating a suspicious bug report. Works backwards from
  symptom to evidence-based verdict — real prod bug, noise, theoretical risk,
  or already protected. Tool-agnostic; assumes you have access to error
  monitoring, logs, analytics, code, and git history.
---

# bug-dig

Most "bug reports" are one of four things. The job of investigation is to
decide which one — quickly, with evidence, without skipping the questions
that drive priority.

## Root cause principle (load-bearing)

The root cause is **always in the system** — never in user behavior. If
your conclusion is "the user did something wrong," that is not a valid
root cause. Reframe: why does the system allow or fail to handle this
gracefully?

A valid root cause must point to a specific line, function, service, or
missing validation in code. "User entered invalid input" is a symptom;
"the form has no client-side validation and the server returns 500
instead of 400" is a root cause.

## The four verdicts

| Verdict | Defining criteria |
|---|---|
| **Real prod bug** | Non-zero affected users, exception reaches user, no architectural defence prevents it |
| **Monitoring noise (tech debt)** | Exception swallowed inside the caller, transaction commits, user flow intact. Low-priority cleanup, not a user incident |
| **Theoretical risk** | Code path is vulnerable but zero production evidence in the lookback window |
| **Already protected** | Real-looking failure that's actually neutralized by some defensive layer (DB constraint, retry, idempotency key, upstream guarantee) |

The verdict drives everything downstream — priority, owner, ticket
shape, communication. Get this right.

## Inputs you accept

Anything is fine — you derive the rest:

- An error-monitoring issue ID
- An existing tracker ticket
- A symptom phrase ("500 at checkout for some users")
- A log snippet
- A code location

Don't refuse to start because you "don't have enough." Start, see what
the evidence does and doesn't show.

## Stage 0 — Sync code before reading

Before reading any handler / DB schema / git log, refresh local clones.
Investigating against stale code produces wrong diagnoses ("this
function doesn't exist" / "this column was removed two weeks ago"). One
git pull beats an hour of misdirection.

## Stage 0.5 — Quick triage (parallel, one round-trip)

Before deep-diving, fire three queries in parallel to get a "scope map":

1. **Error scope** — count of similar issues over 30 days, sorted by
   frequency. Tells you whether this is a single-user blip or
   widespread.
2. **Log scale** — `count_over_time` of the keyword over 24h. Tells
   you whether the log signal matches the error scope.
3. **Entity context** (if you have a user/transaction/task ID) —
   counts by status over 24h. Tells you whether the affected user is
   typical or an outlier.

Read the result as a triangle:

- All three positive → Stage 1 → 5, normal investigation
- Errors but no logs → check for environment-tag mismatch (staging
  events leaking into prod project)
- User reported error but errors=0 and logs=0 → **browser-side path**
  (Stage 2.1)

## Stage 1 — Error monitoring: scope, scale, tags

Look at the issue itself:

- First event — when, who, on what request
- Tags — environment, release, browser, route
- Breadcrumbs — HTTP call ordering, SQL pre/post error, external URLs
- Multiple events across culprits — a shared external dependency can
  produce the same group ID under different handlers; that pattern
  matters

## Stage 2 — Logs: tracebacks and URLs the error monitor truncates

Error monitors group by error message; external URLs and HTML bodies
get truncated. Logs preserve them. Use logs when:

- The error message itself isn't enough to locate the failure
- You need the full request body or response
- The error monitor shows generic exception class (`ClientResponseError`,
  `HTTPError`) — the URL is what disambiguates

Filter aggressively: `!= "<!DOCTYPE"` strips HTML response bodies that
inflate token counts.

## Stage 2.1 — Browser-side errors (errors=0, logs=0, but user reported)

CORS errors, JavaScript fetch failures, download errors happen in the
browser. The backend never sees them, so error monitoring shows nothing
and logs show nothing — but users hit them every day.

Symptoms: user complaint about download/display, but server-side
evidence is silent.

Investigation path:

1. **Product analytics** — search for the related event ("Results
   Download Error" or equivalent). Real scale appears here.
2. **Code** — does the backend return a direct storage URL or a
   proxied URL? Direct storage requires CORS configuration on the
   storage; proxied doesn't.
3. **Frontend** — what mode is the fetch in? `cors` mode requires
   `Access-Control-Allow-Origin` from the storage origin.
4. **Edge cache** — first request without `Origin` header can cache
   a response without CORS headers; subsequent requests fail. CDN
   layer matters.

A CORS bug touches every download flow at once because the affected
code is upstream of all of them.

## Stage 3 — Code review

Read the culprit handler with these specific things in mind:

- **Transaction boundaries.** If money or credits change hands, find
  the commit point. A `try/except` between debit and credit can
  swallow the exception and skip the rollback — leaving an
  inconsistent state.
- **External callers after a commit.** Some clients raise on HTTP
  errors, some swallow them with a warning log. Know which is which.
  An external call that swallows after `commit()` looks like a
  successful flow with a "warning" — but the side effect happened
  while the warning was being logged.
- **Idempotency.** App-level "check before insert" is not enough
  under concurrency. Look for DB `UNIQUE` constraints. If the table
  doesn't have one, race conditions are possible.
- **HTTP wrapper layers.** Many stacks have a wrapper that logs
  request bodies on failure *before* re-raising. The exception ends
  up in the error monitor under the caller's culprit even though the
  caller swallowed it. Pattern: "error in handler, but transaction
  committed fine" → almost always this.

## Stage 4 — Git correlation

Look at deploys around `first_seen`:

```
git log --all --format='%ad %h %s' --date=short --since=<...> --until=<...>
```

A first-seen timestamp coinciding with a `feat: X` commit is the
strongest signal you'll get. Match it, then:

```
git show --stat <hash>
git log --follow <file>
```

The author of the relevant change is the suggested fix owner. Not
because they're at fault — because they have the context.

## Stage 5 — Cross-source enrichment (optional)

- **Analytics** for context — is this primary flow or secondary?
  ("Primary auth works fine, broken secondary can be temporarily
  disabled.")
- **Wiki / docs** — was this service added for a specific reason
  the recent code reviewer doesn't know? Vision docs explain
  surprising architecture.
- **Metrics dashboards** — visual scale for the "is this big?" sanity
  check, when raw counts feel ambiguous.

## Stage 5.5 — AI/ML content investigation (conditional)

Trigger when the bug looks AI-related (mentions of prompts, generation
failures, content blocks, moderation, safety policies; or the failing
code path is in a generation/inference handler).

Collect:

1. **The input** — prompt text, model, provider, parameters.
2. **The user's permissions at the time** — what flags governed
   whether this content type was allowed for this user.
3. **The classifier verdict** — what the safety/moderation layer
   returned. Was it `flagged: true` with a reason? Was it skipped?

The verdict dimension differs from regular bugs:

| Category | What it means |
|---|---|
| **False positive (over-block)** | Classifier blocked legitimate input; tune classifier or whitelist |
| **False negative (under-block)** | Disallowed input passed through; strengthen classifier or add prefilter |
| **User-props mismatch** | User permissions allowed the content but the handler refused — bug in routing logic |
| **Provider-side block** | Classifier passed, but the upstream model/provider rejected — escalate to provider or change provider |
| **Not AI-content** | Triggers were misleading; continue with normal Stage 6 |

Action items differ by category — over/under-block fixes the
classifier; mismatch fixes the handler; provider-block escalates to
the provider.

## Stage 6 — Compose the verdict

The verdict is one of the four. Above the verdict, write the evidence
in this order:

1. **TL;DR** — 1-2 lines, no jargon. What happened, who's affected.
2. **User impact** — yes/no callout. The single most important line;
   it drives priority. Numbers + period + last seen.
3. **Root cause** — concrete location in code or system. Reframe if
   you wrote "user did X."
4. **How to fix** — options, ordered by reversibility. Prefer one
   plan; offer multiple only when fixes are different in nature
   (backend+frontend, hot-fix+architectural).
5. **Sources** — links to the issues, dashboards, code, docs that
   support the verdict.

## Stage 7 — Output, never auto-create tickets

Verdict to chat first. Always. This is the user's instant overview and
the fallback if downstream writes fail.

If a corresponding ticket exists, comment on it with the verdict +
evidence. The comment is not the same shape as the chat output — chat
allows free form; tracker comments need the team's writing conventions
(see `task-comment`).

**Hard rule: never auto-create a tracker ticket from this skill.**
Recommendation goes to chat ("Suggest filing a Bug ticket: priority
Medium, name X. Use `task-create`"). Ticket creation is a separate
deliberate decision, not a side effect of investigation.

If the verdict is for the candidate-bugs list (real bug worth tracking
even before a ticket), delegate to a separate write-step skill that
handles confirmation. Never silently fan out from one investigation
skill to multiple write destinations.

## Anti-patterns

- ❌ "User did X wrong" as root cause
- ❌ Skipping the user-impact callout — that's the priority driver
- ❌ Severity / priority hand-waved into the description instead of
  set as a field
- ❌ Re-using numbers from an old description without re-verification
  (rates change daily; raw error counts ≠ exception events 1:1
  because of different windows and capture points)
- ❌ Implicit scope — "only X" / "all providers" / "only Web" —
  scope is part of user impact
- ❌ Action items written with class names, exception types, file
  paths in the user-facing line. Test: does this read without
  knowing the codebase?
- ❌ Same body for the wiki page and the tracker comment — different
  audiences, different length budgets
- ❌ Multiple bundled findings as "additional notes" in one comment
  — separate findings get separate tickets
- ❌ "Critical" verdict from monitoring noise alone, with no user
  impact
- ❌ Missing deploy correlation when one matches — commit hash +
  author is the fix owner
- ❌ Auto-creating tracker tickets — this skill never writes there
