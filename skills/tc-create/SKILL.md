---
name: tc-create
description: >-
  Methodology for writing test cases that survive review. Covers when to write
  a TC, where its data should come from, how to title and prioritize it, what
  status to assign, and how to format steps. Tool-agnostic — apply with any
  TMS, any tracker, any analytics stack.
---

# tc-create

Writing a test case is two decisions: **what to capture** and **how confident
am I that what I captured is correct**. Tooling matters far less than
getting these two right.

## When to write a TC

You're looking at a behaviour worth capturing if at least one is true:

- It corresponds to a real product event seen in analytics (it actually
  happens).
- It corresponds to a code path you can point at (it's actually
  implemented, not theoretical).
- It corresponds to a real failure observed in monitoring (it has
  actually broken before).
- It's on a critical user path (auth, payment, data integrity, anything
  whose silent failure would matter).

If none of these — you're writing speculation. Stop or downgrade to a note.

## Where the data comes from — never fabricate

Every value in a test case must trace to a real source:

| What | Source |
|---|---|
| Event name, property values | Product analytics |
| Endpoint, status code, payload shape | Code or API spec |
| Error message text | Error-monitoring tool, real captured event |
| UI button label | Screenshot, design file, or running app |
| User states, role flags | Database schema or code |

If you don't know the exact value, write a placeholder (`<button-label>`,
`<error-text>`) and flag it for verification — never guess. A TC with an
invented label fails or passes for the wrong reason and erodes trust in
the whole suite.

## Status — three confidence levels

A test case carries a confidence claim. Be honest about it:

- **GUESS** — source is verified (the event/code/error exists), but the
  steps are not yet walked through. Default for output of bulk gap
  analysis or first-pass writing.
- **DRAFT** — partially verified, work-in-progress. You've executed at
  least once but something is incomplete (steps, expected results,
  edge cases).
- **ACTIVE** — every step has been walked through against the real
  product and matches reality. Promotes only on explicit decision.

When in doubt, drop down a level. `GUESS` over `DRAFT`. Better to
under-promise than to mislead a future reviewer who trusts the status.

## Hard rule: don't quietly modify ACTIVE TCs

ACTIVE means someone trusted it. Modifying silently breaks that trust.
The flow is always: stop, show planned diff, wait for explicit
confirmation. Same applies to bulk operations — the rule scales.

## Title patterns

Three shapes cover almost every TC:

| Shape | Use for | Example |
|---|---|---|
| `EventName: prop=val` | Analytics-driven cases | `Task Created: type=video` |
| `condition → consequence` | Negative / edge cases | `credits=0 → Task Created not fired` |
| `Object: short description` | UI / backend behavior | `Promote action: requires confirmation` |

Title rules:

- Don't repeat the folder/category name in the title — folder context is
  already there.
- No fluff: drop "successful", "correct", "happy path", "valid". The
  status field already says whether it's positive or negative.
- Analytics names stay in the original language they're emitted in
  (usually English with `prop=value` notation). Description text in your
  team's working language.
- Title fits one line. If it doesn't, you're describing two cases.

## Priority — anchor in volume, override on criticality

Use real numbers from analytics, not gut feel:

| Priority | Volume / 30 days | When |
|---|---|---|
| HIGH | >50K events | High-traffic flow |
| MEDIUM | 5K–50K | Mainstream but not dominant |
| LOW | <5K | Rare, edge case |

Override volume when the path is critical: auth, payment, account
deletion, data export. Low-volume but unrecoverable failure modes are
HIGH regardless of count.

## Steps format

```
action → expected
```

- `action` is imperative. Not "user clicks", just "click X".
- `expected` is the observable result, not a restatement of the action.
  "Click Submit → success message appears" is fine. "Click Submit →
  Submit button is clicked" is noise.
- One outcome per line. If a step produces two checks, split.
- UI labels come from real screenshots, not memory. Placeholder + flag if
  unsure.
- Don't number unless the order matters across the whole flow. Most TCs
  are sequential by default.

## Duplicate check (when working in bulk)

Before creating a new TC, check existing TCs in the same project for
overlap. A simple algorithm works:

- Lowercase, strip punctuation, split into words.
- Drop stopwords — filler conjunctions in your working language, plus
  symbols like `→`, `:`, `=`.
- Count significant-word overlap with each existing title.
- ≥ 2 significant words shared = potential duplicate, surface for review.

Cross-project overlaps are usually intentional (Web TC mirrored as Back
TC). Same-project duplicates need discussion: keep both, merge, skip,
update.

If the existing TC is `ACTIVE` — double-confirm before any modification.

## Modes (orthogonal)

Two axes, four combinations:

|  | **Single** (one at a time) | **Bulk** (list upfront) |
|---|---|---|
| **Direct** (you describe the gap) | Interactive write, confirm, create | Each title gets duplicate check |
| **Triage queue** (skeletons exist) | Pick one skeleton, research, upgrade | Process N skeletons, batch upgrade |

Triage queue assumes a separate gap-analysis pass has produced skeleton
TCs that point at *what* to test without claiming *how*. The `tc-create`
methodology fills in the *how* — adding steps and bumping status from
something like "skeleton" to `GUESS`/`DRAFT`.

## Anti-patterns

- ❌ Inventing UI labels or error text from memory
- ❌ Writing TCs for code paths that don't exist yet ("we should test
  X feature when we build it") — that's a planning note, not a TC
- ❌ Promoting to `ACTIVE` to clear status without walking steps
- ❌ Folder-name in title (`Auth: Auth via email...`)
- ❌ Fluff adjectives that don't add information
- ❌ Speculative volume — "probably high traffic" — when analytics is
  available
- ❌ Modifying ACTIVE without confirmation
