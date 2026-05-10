---
name: task-create
description: >-
  Methodology for writing bug tickets that read well to both product and
  engineering. Two-layer structure (Product layer / Engineering layer),
  required sections per bug type, priority logic with downstream-effect
  awareness, and writing rules that survive review. Tool-agnostic.
---

# task-create

A bug ticket has two readers — product and engineering — with very
different needs. The methodology splits the body into two layers so
neither audience has to dig through the other's content. Above that,
naming, priority assignment, and writing style decide whether the
ticket gets read at all.

Hard rule: **always show a draft and wait for explicit confirmation
before creating or updating a ticket.** Apply at the skill boundary,
no exceptions.

## Two templates: bug vs initiative

Pick by signal:

| Signal | Template |
|---|---|
| Reactive fix on a specific symptom, one PR, one author | **Bug** |
| Tech-debt with one concrete fix | **Bug** |
| Multi-stage QA initiative (test automation, regression suite, cross-team coordination) | **Initiative** |
| Research task with multiple investigation threads | **Initiative** |
| Phased infrastructure cleanup (PoC → MVP → expansion) | **Initiative** |
| Existing initiative accumulated progress and needs a current-state snapshot | **Initiative (update mode)** |
| Atomic event in an existing ticket without body change ("phase done", "request sent") | Comment, not this skill |

The rest of this document covers the **bug** template. Initiative is a
different shape (TL;DR + Scope + Plan + What's added + Progress
append-only) and is its own pattern; same hard rule about confirmation
applies.

## Step 0 — Duplicate check (skip only when user gave a specific URL)

Before creating, search the tracker for existing tickets:

1. Pull 2-3 keywords from the bug — exception name, feature, provider,
   endpoint
2. Search same project, show top 3 matches with name + URL + status
3. If similar exists — ask: comment on existing, or new ticket
4. "Comment" → switch to comment mode
5. "New" → continue to Step 1

Don't over-query — 1-2 targeted searches. >5 results means you're too
broad; tighten the keywords.

## Step 1 — Title

User-symptom oriented, no jargon. Pattern: `<what user sees> — <where/when>`.

- **What user sees** — product symptom, understandable without code
  knowledge (mandatory)
- **Where/when** — screen, feature, brand, condition (optional, only
  if not obvious from "what")

Examples:
- `Registration with banned email domain: system error instead of clear refusal`
- `<brand>: blocked tools aren't hidden for anonymous users`
- `<provider> checkout: user doesn't see confirmation after payment`

Forbidden in title:
- Bug-tracker prefixes (`[BUG]`, ticket IDs), emoji
- HTTP codes (`401`, `500`, `403`)
- API paths (`GET /api/v1/...`)
- Class / exception names (`NoSuchKey`, `BanReason enum mismatch`)
- Technical jargon a product reader won't decode

## What-Where-When rule for numbers in body

Every number needs:

- **What** — which metric / symptom
- **Where** — endpoint / provider / screen
- **When** — time window ("over 34 days", "since 2026-03-24" or
  `first_seen — last_seen`)

**Rule: a number without a time window is incomplete.** Always attach
the period. Default to `first_seen — last_seen` from error monitoring.

## Two-layer body

The body is two independent modules separated by a horizontal rule:

- **Product layer** — what the user sees, scope, expected vs actual,
  team action items. Read by product.
- **Engineering layer** — where it breaks in code, why, acceptance
  criteria. Read by engineering.

New tickets get both. Comments can carry one or both depending on
context.

## Product layer — required and optional sections

### TL;DR (required)

User symptom + cause-effect chain in plain product language. Template:

> `<user symptom>. Because <root cause in product terms>, <observable consequence>.`

Forbidden in TL;DR:
- Numbers (those go in User Impact and Evidence)
- HTTP codes
- Class / function names
- The word "critical" / "блокер" — priority is a field, not body
  content
- Speculation about why users feel things ("they lose trust", "they
  leave silently")
- Parenthetical explanations of internal product names ("(our brand
  for X)") — internal audience knows them

### User Impact (required)

Value: `yes / no / possible / unknown`.

- **yes** → numbers here. N events / N users over period + last seen
  + what users lose. Three mandatory components: a number, a time
  window, a last-seen date. Missing one → don't show the draft, get
  the data first.
- **no** → if "no" says everything, no further explanation needed.
  Just the heading and the value.
- **possible** → under what scenario it would become real; why we
  don't yet know.
- **unknown** → what to check to find out.

User Impact is the single most load-bearing section. It drives
priority. Get the numbers right.

### Steps to reproduce (optional, mark `[claude-analysis]` if derived)

Skip if no meaningful steps known. When user-visible — UI steps. When
business signal only (analytics) — analytics-tool steps in product
layer; logs / monitoring / metrics steps in engineering layer.

Mark `[claude-analysis]` if derived from stack trace / logs without
manual reproduction.

### Expected vs Actual (optional)

Skip when:
- User Impact = no (user sees nothing)
- User Impact already describes the gap (e.g. "user waits 60 sec
  instead of 5" — that's expected/actual in one phrase)

Otherwise, fill it. For complex bugs with minimum-vs-ideal fix
levels, this section can carry a product-level action item ("on
fingerprint failure UI must not lose state; on data wipe show clear
re-login message"). Engineering-level acceptance criteria stay in
the engineering layer.

### Action items (required)

Imperative phrasing or "must / should". Product language; no class
names, exception types, or file paths. Test: can a non-engineer read
this and know what they want done?

**Level labels are optional.** Add only when items are *different in
nature*. Test: "is this one PR by one person in one sitting?" Yes →
no labels. No → label by the axis along which items differ.

Three legitimate axes (combinable with `/`):

| Axis | When | Labels |
|---|---|---|
| Platform / layer | Changes in different repos / teams | `Backend` / `Frontend` / `Mobile` / `Admin` |
| Solution maturity | Same symptom, different fix depth based on time budget | `Hot-fix` / `Proper fix` / `Architectural` |
| Mandatory-ness | Some items aren't required to close the ticket | `Required` / `UX` / `Optional` |

### Product risk (optional, non-obvious only)

The trap is filling this section with truisms. The rule: **only the
non-obvious belongs here.**

Bad (everyone already knows):
- "Loss of analytics is bad"
- "Longer it stays unfixed, bigger the hole"
- "Users may leave"

Good (non-trivial connections):
- "Error swallowed → monitoring doesn't alert; only found via
  weekly review; the same swallow-pattern in any future integration
  will hide just as silently"
- "Related break: list_all in admin fails with the same LookupError"
- "449 events drown the error budget and mask other 500s"

If nothing non-obvious applies, skip the section.

### Section table by bug type

| Bug type | TL;DR | User Impact | Steps | Expected/Actual | Action items | Risk |
|---|---|---|---|---|---|---|
| User sees an error | ✅ | ✅ yes + numbers | ✅ if known | ✅ | ✅ | ✅ if non-obvious |
| Business analytics broken, user sees nothing | ✅ | ✅ no, no extra explanation | ✅ analytics steps in product, logs in engineering | ❌ skip | ✅ | ✅ if non-obvious |
| Tech debt / monitoring noise | ✅ | ✅ no, no extra explanation | ✅ if any | ❌ usually skip | ✅ | ✅ why fix anyway |
| User sees something, impact unclear | ✅ | ✅ possible + what to check | ✅ if known | ✅ | ✅ | ✅ if any |

## Engineering layer — required structure

```
Where it breaks
- file/path:line

Possible root cause
[code-derived analysis, marked as analysis (not manually verified).
Which line raises, which guard is missing. 3-5 line code snippets ok.]

Acceptance criteria
- [what "done" means — what error disappears, what user sees instead]

Evidence
- Error monitor: <link> — N users, N events, timeframe
- Logs: <link>
- Analytics: <link>
- (Whatever sources the investigation drew on)
```

### Optional engineering sections

| Section | When |
|---|---|
| `Traceback` | Stack trace explains better than prose |
| `Deploy correlation` | Issue first-seen matches a commit — include hash + author + date |
| `Scope` | Narrow impact (one endpoint, one user group, staging only) |
| `Why this still matters` | User Impact = no but downstream harm exists (skip if it just duplicates Product Risk) |
| `Risks of fix` | Proposed fix may break X — area + assessment + explanation |

### `[claude-analysis]` disclaimer

Mark sections in the **product layer** that were derived from stack
trace / code without hands-on verification:

- Steps to reproduce — almost always `[claude-analysis]`
- Expected vs Actual — `[claude-analysis]` if "Actual" was inferred,
  not browser-verified

Engineering layer is by definition derived analysis — no per-section
marking needed there.

## Step 3 — Type and Priority

**Type:** Bug / TechDebt / Enhancement / Feature / Research / QA.
Pick by what the work *is*, not what triggered it.

**Priority:**

| Priority | When |
|---|---|
| Critical | Production broken for many / revenue loss |
| High | Real user pain (direct OR via downstream effects), reproducible, significant reach |
| Medium | Real issue, limited reach or workaround exists |
| Low | No user impact, noise, tech debt |

### High via downstream effects (not just direct user pain)

A bug with no direct user-visible symptom but serious team consequences
is **High**, not Medium. Markers:

- **Long unnoticed** — bug lived >1 week before discovery → no guard
  exists for this kind of thing
- **Confirmed damage** — something concrete already doesn't work or
  can't be done (investigation, on-call, capacity planning, release).
  Not hypothetical.
- **Broken protection** — alerts / monitoring / tests / observability
  not working → **higher chance of missing future user-visible
  incidents**. High even without direct symptom.
- **Wide reach across teams** — affects >1 team or >1 working
  process, even if the user sees nothing.

Test: "if we leave it for another week, will something significant
break?" Yes → High. "Will be inconvenient but works" → Medium.

**Anti-pattern:** defaulting to Medium just because "user doesn't see
it." Direct user pain is *not* the only High criterion.

## Writing style

- Short, direct sentences without padding
- Facts inline: numbers, dates, paths in prose where they fit
- Cause-effect via "because", "since", "to" — connect observation
  with cause in one sentence
- "We"-language for code and decisions: "our exception handler", "we
  don't log this"
- Refer to users by role / count, not email: "the user from the
  ticket", "5 users"
- Don't explain what code does — explain why it causes the problem
- Don't explain internal product/brand names parenthetically — the
  audience knows them
- Product layer is jargon-free. HTTP codes, class names, file paths
  belong in engineering layer.
- Numbers go only in User Impact and Evidence. Not in TL;DR or Risk.
- Don't invent risks. Only what genuinely follows from this bug.
- Neutral tone about code and team decisions. Describe consequences
  or facts, not judgments. Bad: "naive fix". Good: "fix that
  addresses one bug without considering the other will cause issues."
- Impersonal facts > "I"-action. "I found 10 cases" → "10 cases
  exist". "I checked that..." → "logs show...". Reader doesn't care
  who specifically found it.
- No filler adjectives. "Known latent bug" → "latent bug". "Known",
  "new", "small", "fairly large" carry no information; reader can
  click through if curious.

## Anti-patterns

- ❌ Skip Step 0 (duplicate check)
- ❌ Write to tracker without confirmation
- ❌ Severity / priority hand-waved into description body — fields
  exist for that
- ❌ HTTP codes, paths, class names in title
- ❌ TL;DR with technical jargon — product reads TL;DR
- ❌ Parenthetical explanations of internal product names
- ❌ Numbers in TL;DR or Risk (only User Impact and Evidence)
- ❌ User Impact buried inside TL;DR — must be its own section
- ❌ Speculative user-behavior claims in TL;DR or Actual ("part of
  users leave without retrying", "loses trust") — write only what
  the user *sees / does*
- ❌ Philosophical risks ("constant background noise", "errors will
  always happen") — generic, not specific
- ❌ Repeating the same idea in different words inside one Risk
  bullet
- ❌ Obvious risks in Product Risk ("losing analytics is bad")
- ❌ Filling optional sections for completeness — skip if nothing
  meaningful
- ❌ Duplicating Expected/Actual with User Impact
- ❌ Judgmental adjectives about team work ("naive", "wrong",
  "should have") — neutral and factual
- ❌ Long explanation under "User Impact: no" when "no" says
  everything
- ❌ Engineering layer above Product layer
- ❌ "Critical" word in TL;DR — it's a field, not a phrase
- ❌ User Impact = yes without N events/users or last seen — don't
  show the draft, gather data first
- ❌ Action items written with class names / exception types / file
  paths — those go in `Where it breaks` / `Possible root cause`,
  not in the user-facing item line
- ❌ Level labels (`Backend / Hot-fix / Optional`) on items of one
  nature — that's a menu, not a plan
- ❌ Action items without level labels when items are different in
  nature
- ❌ A separate related issue with its own user impact stuffed as
  "additional finding" in one ticket — separate findings get
  separate tickets/subtasks
- ❌ Default Medium for a bug with no direct user pain — apply the
  downstream-effects test first
- ❌ Missing Evidence section
- ❌ "I"-form in comments ("I found", "I checked", "I want to
  observe") — rewrite impersonally
- ❌ Filler adjectives without information
- ❌ Self-announcing admin operations ("I'll log this separately
  later") — promised work isn't visible; just do it and let it show
- ❌ Mentioning the ticket's own ID inside a comment on that
  ticket — tautology
