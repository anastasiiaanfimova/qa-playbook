---
name: ui-snapshots
description: >-
  Run, extend, or maintain the <product> UI screenshot catalog at
  <product-dir>/ui-snapshots — repeatable Playwright snapshots of every page
  in the <product> staging frontend (desktop + mobile, light + dark, with
  per-property user-state combinations). Also opens the local HTML viewer
  (faceted gallery with viewport/theme toggles and per-property filters).
  Subcommands: `/ui-snapshots view` (default — open the gallery, ~2 sec),
  `/ui-snapshots run` (full Playwright refresh ~20 min, ASK first),
  `/ui-snapshots state <label>` (ad-hoc DB toggle outside the runner).
  Triggers: "/ui-snapshots", "/ui-snapshots view", "/ui-snapshots run",
  "ui-snapshots", "ui снапшоты", "снапшоты сайта",
  "screenshot пробег <product>", "новые скрины", "обнови снимки сайта",
  "добавь страницу в снапшоты", "сними под другим состоянием юзера",
  "state-overrides snapshot", "открой вьюер", "посмотреть снимки",
  "открыть viewer", "открой галерею снимков", "посмотреть скрины",
  "обнови и открой вьюер".
---

## Subcommands

`/ui-snapshots <subcommand>` — explicit mode selection.
Without subcommand the **default is `view`** (the most common, safe action).

| Subcommand | Action | Time | When |
|-----------|--------|------|------|
| `view` (default) | `npm run viewer:open` | ~2 sec | "open viewer", "посмотреть снимки", "открой галерею" |
| `run` | full Playwright refresh via runner | ~20 min | "обнови снимки", "новые скрины", "сделай свежие снапшоты" |
| `run --since <ref>` | partial refresh based on frontend git diff | ~5 min typical | "обнови затронутые страницы", "снапшоты только для ветки" |
| `state <label>` | ad-hoc DB toggle (outside runner) | ~5 sec per toggle | "сними под другим состоянием юзера", "state-overrides snapshot" |

### `/ui-snapshots view` — safe, fast, always

```bash
cd <product-dir>/ui-snapshots && npm run viewer:open
```

This **only regenerates `output/viewer.html`** from existing captures and opens
it in the default browser. Does NOT run Playwright, does NOT take new
screenshots, does NOT touch staging. Takes 1-2 seconds.

**Trigger this for any "view/open/look at" intent.** No need to ask permission —
it's idempotent and cheap. Use as default when intent is unclear.

### `/ui-snapshots run` — full refresh (long, ASK FIRST)

```bash
cd <product-dir>/ui-snapshots
INFI=/Users/<user>/.claude/scripts/infisical-<product>-mcp.sh

# Plan only (no DB writes, no captures):
$INFI npm run runner:dry

# Full refresh — 1148 captures across 4 viewport×theme variants, ~20 min:
$INFI npm run runner

# Subset:
$INFI node capture/runner.js --groups settings_credits,billing_credits
$INFI node capture/runner.js --archetype admin
$INFI node capture/runner.js --viewports desktop --themes light

# After completion:
node utils/index.js
git add -A && git commit -m "Refresh: $(date +%Y-%m-%d)" && git push
npm run viewer:open
```

**Always ask the user to confirm before running** — it's ~20 min and may hit
staging rate limits. Confirm whether they also want commit+push to history
repo at the end.

Live progress at `output/_progress.json` (heartbeat every 10 captures).
Failures don't abort the run; baseline DB state always restored on exit.

### `/ui-snapshots run --since <ref>` — selective rerun

Запускает runner только по группам, которые могли измениться, по `git diff` во `frontend/` репо vs `<ref>`. Типичная экономия: 20 мин → 5 мин.

```bash
cd <product-dir>/ui-snapshots
INFI=/Users/<user>/.claude/scripts/infisical-<product>-mcp.sh

# Plan only (без захвата) — вернёт список групп или FULL_REQUIRED:
node capture/affected-slugs.js --since main --frontend-repo <product-dir>/frontend

# Запуск только затронутых групп:
$INFI npm run runner -- --since main

# Если global/shell файлы тоже затронуты — без флага скрипт упадёт с подсказкой:
$INFI npm run runner -- --since main --auto-full   # запустить полный
$INFI npm run runner -- --since main --auto-skip   # пропустить uncertain/full, гнать только confident
```

**Как работает mapper (`capture/affected-slugs.js`):**

| Категория | Files matching | Действие |
|-----------|----------------|----------|
| `page` | `frontend/src/app/<route>/page.tsx`/`layout.tsx` | URL → STATIC_ROUTES → slug → groups из page-property-map.json |
| `global` | root `app/layout.tsx`, `globals.css`, `styles/`, `providers.tsx`, `tailwind/postcss/next.config.*`, `package.json`, lockfile | full required |
| `shell` | `components/(layout\|nav\|sidebar\|header\|footer\|shell)/*` | full required |
| `lib` | `lib/api/*`, `hooks/*` | full required |
| `uncertain` | `components/<other>/*` | вывести список, пользователь решает |
| `ignore` | `*.test.tsx`, `*.spec.tsx`, `__tests__/`, `*.md`, `.gitignore`, `.eslintrc*` | пропустить |

**Когда использовать:** в `branch-analyze` после `git diff` ветки; после своих локальных правок во frontend перед commit для swift visual check.

**Когда НЕ использовать:** перед релизной полной верификацией (риск пропустить cross-cutting эффект из uncertain компонентов).

---

### `/ui-snapshots state <label>` — ad-hoc DB toggle

For one-off DB state changes outside the runner (debugging a single page,
manual investigation):

```bash
$INFI node capture/snapshot-state-toggle.js get             # read current state
$INFI node capture/snapshot-state-toggle.js set paying-no   # toggle
$INFI node capture/snapshot-state-toggle.js restore         # restore baseline
```

For systematic per-state captures use the runner — `page-property-map.json`
declares which properties affect which slugs, runner generates the combos.

### Disambiguation

- "открой вьюер" / "посмотреть снимки" → `view` (NOT a refresh)
- "обнови снимки" / "новые скрины" → `run` (CONFIRM first)
- "обнови и открой вьюер" → ambiguous — ask: "viewer regen (2 sec) or full
  refresh (20 min)?" Default to viewer-regen if user shows impatience.

## Source of truth in MemPalace

ALWAYS load these before doing real work — they hold current state and
pending TODOs:

```js
mempalace_search("ui-snapshots structure", wing="<product>")
// → drawer_<product>_tools_872896282d96e0cc2b3d4cf5  (architecture)
// → drawer_<product>_tools_79b69f6773a229995522bfad  (TODO + pending work)
```

The architecture drawer has the runner internals, file naming, env vars,
common gotchas. The TODO drawer has whatever wasn't finished last time.

## Quick orientation

```
<product-dir>/ui-snapshots/
  package.json              npm scripts: runner, runner:dry, viewer, viewer:open,
                            snapshot, login, index, bootstrap-users, state-toggle
  capture/                  snapshot engine
    runner.js               unified capture (replaces all old runners)
    snapshot.js             single-URL CLI: node capture/snapshot.js URL [slug]
    snapshot-state-toggle.js  ad-hoc DB toggle (outside runner)
    snapshot-helpers.js     takeFullSnapshot() — full-page capture
    bootstrap-test-users.js idempotent INSERT/UPDATE for test users
    interaction-actions.js  24 interaction sequences + executeAction helper
  viewer/                   HTML gallery
    viewer.js               build output/viewer.html
    viewer-data.js          walk output/ and index captures per slug
    viewer.runtime.js       browser runtime inlined into viewer.html
    viewer.css              styles inlined into viewer.html
  shared/                   used by capture + viewer + utils
    state-id.js             encode/decode state-id; archetype defaults; combos
    categories.js           slug → output/<NN-section>/ mapping
    states-manifest.js      builds output/states.json from captured directories
    routes.js               STATIC_ROUTES + VIEWPORTS
    dynamic-routes.js       DYNAMIC_ROUTES — [param] pages with real IDs
    auth-helpers.js         inlineLogin(page) + safeNavigate(page, url)
    test-users.js           archetypes: user / admin / blocked / not-user
    page-property-map.json  per-page property-relevance map (runner's plan input)
  utils/                    one-off and verification scripts
    index.js                generates output/index.md + per-section indexes
    login.js                standalone login → auth/storage-state.json (legacy)
    verify-login.js         smoke-check: can each archetype log in
    verify-is-paying-billing.js  ad-hoc: confirm is_paying actually changes UI
  auth/                     storage-state.json (gitignored)
  output/                   captures + viewer.html (committed history)
    states.json             manifest — list of captured states per archetype
    viewer.html             single-file gallery
    _progress.json          live runner progress (gitignored)
    01-public-landing → 12-admin-workflows + 99-other
      state-<id>/           PNG + MD per state
        <viewport>-<theme>--<slug>.{png,md}
```

**File path inside any section:**
`output/<NN-section>/state-<id>/<viewport>-<theme>--<slug>.{png,md}`.

State-id encodes the full 8-property user tuple — self-describing, no implicit
defaults. Example: `state-blocked.no_credits.high_is-paying.yes_..._role.user`.

Greps you'll use:
```bash
find output -name "desktop-light--*.md"           # all desktop-light cards
find output -name "*--settings.png"               # settings across all variants
ls output/04-account/state-*/                     # all states for /settings section
```

## Critical lesson — never use storageState

Playwright's `storageState` save/restore drops `__Host-session` cookies because
that prefix forbids a `domain` attribute (which storageState writes anyway).
Long-running batches that load `auth/storage-state.json` from a previous
`login.js` run silently render as **anonymous** — pages 200 but UI is the
public version. We were burned by this on the overnight run 2026-05-01.

**Always use `inlineLogin(page)` from `auth-helpers.js`.** runner.js already
does this — don't introduce storageState in any new code.

Verify after a run: open one MD card and check that user-specific content
(credits balance, archetype-specific UI) renders.

## Critical lesson — safeNavigate over networkidle

`waitUntil: 'networkidle'` is too strict. Pages that poll task status (e.g.
`/my-generations`) never reach idle and time out.

**Use `safeNavigate(page, url)` from `auth-helpers.js`:**
- `waitUntil: 'load'` (HTML + images + CSS, but not async data)
- best-effort `waitForLoadState('networkidle', { timeout: 10s })` — won't throw if missed
- 3-second `waitForTimeout` after for React hydration
- 45s overall timeout

runner.js uses this everywhere. Don't call `page.goto` directly.

## The runner — runner.js

Single entry point for all bulk captures. Reads `page-property-map.json` to
decide which user-state combinations matter for each slug, then iterates
groups × slugs × per-page combos × viewport × theme.

### CLI flags

| Flag | Purpose |
|------|---------|
| `--dry-run` | plan only — no DB writes, no captures |
| `--groups g1,g2` | subset by group name (`settings_credits`, `tool_result`, etc.) |
| `--archetype user\|admin\|blocked\|not-user` | only this archetype |
| `--viewports desktop\|mobile` | one or both, comma-separated |
| `--themes light\|dark` | one or both, comma-separated |

### What runs in a full pass

1. Plan: build task list from page-property-map.json + filters
2. Per archetype:
   - Connect to Postgres, capture baseline state
   - Per (viewport, theme) variant:
     - Launch Chromium, login once (inlineLogin)
     - Per state-id: set DB state, reload page, snap each slug in that state
   - Restore baseline state in `finally`
3. Write live progress to `output/_progress.json`
4. Final summary with failure list

### Adding a new page or state combination

1. **Code-audit first.** Don't trust intuition — grep frontend code for the user
   property and find every use-site:
   ```bash
   grep -rn "user\.\?<prop>\|user.<prop>" frontend/src --include="*.tsx" --include="*.ts" \
     | grep -v "test\|spec\|<analytics>\|trackUser\|console\|api/mutations"
   ```

2. **Update `page-property-map.json`.** Either:
   - Add a new slug to an existing group (if it shares the same `affectedBy` props)
   - Or create a new group with its own `affectedBy: [...]` and `slugs: [...]`

3. **For new pages, also add to `routes.js` STATIC_ROUTES** (or DYNAMIC_ROUTES
   if it has `[param]` segments).

4. **Dry-run to verify the plan:**
   ```bash
   $INFI npm run runner:dry
   # Check the new slug appears with the right state combos
   ```

5. **Smoke run subset:**
   ```bash
   $INFI node capture/runner.js --groups <new-group> --viewports desktop --themes light
   ```

6. **Visual check** — open one MD/PNG to confirm.

7. **Full refresh + commit + push** (see History below).

## When something looks wrong

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| All snapshots look like landing page | Anonymous render — storageState lost __Host- cookie | Switch to `inlineLogin(page)` |
| Many failures with timeout | networkidle on slow/polling pages | Use `safeNavigate(page, url)` not `page.goto` |
| State toggle doesn't show in PNG diff | Frontend cache via redux-persist OR session lost | runner.js does `page.reload()` between state batches; verify with `mcp__postgres__query` that DB actually changed |
| Dark theme looks identical to light | localStorage theme key not respected | Check next-themes config in `frontend/src/app/providers.tsx` — `forcedTheme` may override |
| `/my-generations` modal returns same as page | Empty test account — no thumbnails to click | Requires test data (3-9 credits to generate). See TODO drawer item B1 |
| Interaction snapshot didn't open dialog | Selector miss (action was `optional: true` → silent skip) | Inspect MD card "Actions executed" section; update selector in `interaction-actions.js` |

## Things that are NOT in this catalog

- **Admin app** (`adm.zncr.pro`) — VPN-only or different host. Currently 404
  via plain web. User said "не трогаем" 2026-05-01.
- **Production** (`<product>.pro`) — separate creds, not in scope.
- **Generation viewer modal upsell** — needs real generations on the test
  account. See TODO drawer section B.

## After every meaningful change to the catalog

1. Run `node utils/index.js` to refresh `output/index.md` and per-section indexes.
2. Run `node viewer/viewer.js` to rebuild `output/viewer.html`.
3. **Commit + push to history repo** so the change becomes part of the design history (see "History" below).
4. Update the architecture drawer if a new file / pattern / lesson appeared:
   `mempalace_update_drawer(drawer_id="drawer_<product>_tools_872896282d96e0cc2b3d4cf5", content="...")`.
5. Update the TODO drawer (`drawer_<product>_tools_79b69f6773a229995522bfad`) — mark items done, add new gaps.
6. Write a short diary entry — `mempalace_diary_write(agent_name="claude-<product>", topic="ui-snapshots-...", entry="AAAK summary")`.

## History — design timeline

The repo `<product-dir>/ui-snapshots/` is a private GitHub repo:
**`anastasiiaanfimova/ui-snapshots-<product>`** (private). Every catalog
refresh becomes a commit; that commit IS the design baseline at that point
in time.

### After a refresh — always commit

```bash
cd <product-dir>/ui-snapshots
git add -A
git commit -m "Refresh: $(date +%Y-%m-%d)"
git push
```

**Critical:** without commit + push, the history is lost on the next refresh.
This is the ONLY way "what did /settings look like 2 weeks ago" works.

### Find the historical version of a page

CLI:
```bash
# Find the path (state-id varies):
ls output/04-account/*/desktop-light--settings.png

# Then:
git log --oneline --follow <path>
git show <commit>:<path> > /tmp/before.png
open /tmp/before.png
```

Browser (often easier — no clone needed):
```
https://github.com/anastasiiaanfimova/ui-snapshots-<product>/commits/main/<path>
```
Click any commit → see that historical version inline.

### Why design-history matters

When picking up a redesign task without strong memory of the page, the first
move is: open `https://github.com/.../commits/main/output/<NN>/state-<id>/<slug>.png`,
see how it looked **before** the design change started. Compare with the new
mock — only then do you know what's *actually* changing.
