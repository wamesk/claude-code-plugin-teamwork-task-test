---
name: teamwork-task-test
version: 1.0.1
description: "Use when the user provides a Teamwork.com URL (single task or tasklist) and asks to 'test these tasks', 'otestuj tasky z teamworku', 'preveruj zadanie', 'skontroluj akceptačné kritériá', 'spusti testy pre tasky', or invokes '/teamwork-task-test'. Fetches tasks via the Teamwork REST API v3 (reusing the shared config/token from the `teamwork-task` plugin), parses acceptance criteria from the task description, detects the project's test stack (Pest, PHPUnit, Laravel Dusk, Cypress, Playwright, Selenium, Vitest, Jest), tries to map criteria onto existing tests and run them, optionally drives a real browser via the chrome-devtools MCP for visual verification, and writes a per-task report with each acceptance criterion individually marked as ✅ verified by test / ⚠️ partial / ❌ failed / 📋 manual / 📝 missing-or-proposed. Never modifies the Teamwork task or board by default — read-only and idempotent unless the user explicitly opts into commenting or board moves. Always use this skill when the user wants to verify or QA work captured in a Teamwork task without having to write or run the tests by hand themselves."
argument-hint: "<teamwork-url> [--run-tests=auto|never|always] [--visual=auto|browser|skip] [--comment-on-task=true|false] [--time-log=true|false] [--language=sk|en]"
allowed-tools: [Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion, WebFetch]
---

# Teamwork Task Tester

Fetch tasks from Teamwork.com (single task or whole tasklist), extract their
acceptance criteria, **try to verify each criterion** against the current
repository — by running existing unit / feature / browser tests where possible,
by driving a real browser via the chrome-devtools MCP for visual checks where
appropriate, and by writing **manual test scenarios** for everything that
cannot be checked automatically. The output is a structured report where every
acceptance criterion is individually annotated with its verification status and
the evidence (test name, file:line, browser screenshot path, or manual
scenario reference).

The user invoked this skill with: `$ARGUMENTS`

This skill is **read-only against Teamwork by default** — it does not move the
task on the board, does not complete it, and does not post a comment unless the
user explicitly opts in. The only thing that *is* written back by default is a
time log (because QA is real work and should appear in the timesheet) — and
even that can be turned off with `--time-log=false`.

---

## Arguments

Expected first positional argument: a Teamwork.com URL pointing to either a
tasklist or a single task. Same URL shapes as the `teamwork-task` plugin.

Recognized URL shapes:
- Single task: `https://<workspace>.teamwork.com/app/tasks/<taskId>` (also `/#/tasks/<id>`, `/tasks/<id>`)
- Tasklist: `https://<workspace>.teamwork.com/app/tasklists/<tasklistId>` (also legacy `/tasklists/<id>`)

Optional flags (override config for this run only — not persisted):
- `--run-tests=auto|never|always` — whether to try to actually execute discovered tests (default: `auto`)
- `--visual=auto|browser|skip` — whether to use the chrome-devtools MCP / a browser test runner for visual criteria (default: `auto`)
- `--comment-on-task=true|false` — post the final report as a comment on each Teamwork task (default: `false`)
- `--time-log=true|false` — write a time log to Teamwork for the QA work (default: `true`)
- `--language=sk|en` — language for the report and any AC suggestions (default: from config, fallback `sk`)

If `$ARGUMENTS` is empty or contains no URL, ask via **AskUserQuestion** for the URL before doing anything else.

---

## Step 1 — Parse the URL

Identical parser to the `teamwork-task` plugin:

```bash
URL="<the url>"
WORKSPACE=$(echo "$URL"   | sed -nE 's|https?://([^.]+)\.teamwork\.com/.*|\1|p')
ENTITY_ID=$(echo "$URL"   | sed -nE 's|.*/(tasks|tasklists)/([0-9]+).*|\2|p')
URL_KIND=$(echo "$URL"    | sed -nE 's|.*/(tasks|tasklists)/[0-9]+.*|\1|p' | sed 's/s$//')
BASE_URL="https://${WORKSPACE}.teamwork.com"
```

If any of the three is empty, ask the user via **AskUserQuestion** to confirm or paste a corrected URL.

---

## Step 2 — Load shared config (reuse `teamwork-task` plugin config)

This skill deliberately shares the config file with the `teamwork-task` plugin,
so users who already configured one have nothing extra to do.

```
~/.claude/plugins/data/teamwork-task-wamesk/config.json
```

Algorithm:
1. `CONFIG_DIR="$HOME/.claude/plugins/data/teamwork-task-wamesk"` — `mkdir -p "$CONFIG_DIR"`.
2. `CONFIG_FILE="$CONFIG_DIR/config.json"`.
3. If `CONFIG_FILE` does not exist:
   - Prefer copying from the `teamwork-task` plugin's bundled template if it is installed (look at `${CLAUDE_PLUGIN_ROOT}/../teamwork-task/config.example.json` or `$HOME/.claude/plugins/cache/.../teamwork-task/config.example.json`).
   - Otherwise copy this plugin's own template from `${CLAUDE_PLUGIN_ROOT}/config.example.json`.
   - `chmod 600 "$CONFIG_FILE"`.
4. Validate with `jq . "$CONFIG_FILE" >/dev/null`. If invalid → report and stop.
5. **First-run check** — if `.teamwork.base_url` or `.teamwork.api_token` are missing/empty/`<workspace>` placeholder, prompt the user via **AskUserQuestion** for both values (prefill `base_url` from the parsed URL). Write them back atomically with `jq` + `mv`, then `chmod 600`.
6. **Apply config migration / defaults for this skill's own keys** — see Step 2.5 below.
7. **Apply CLI flag overrides** (`--run-tests`, `--visual`, `--comment-on-task`, `--time-log`, `--language`) to the in-memory config — do not persist.
8. **Never echo the API token.** Always pass auth to `curl` via `-u` (kept out of `ps`), never in the URL.

### Step 2.5 — Test-skill config defaults (idempotent merge)

Add this skill's own keys to the same config file, defaulting any missing keys:

```bash
jq '
  (.test_skill //= {}) |
  (.test_skill.run_tests //= "auto") |
  (.test_skill.visual_mode //= "auto") |
  (.test_skill.comment_on_task //= false) |
  (.test_skill.time_log //= true) |
  (.test_skill.move_on_pass //= "") |
  (.test_skill.move_on_fail //= "") |
  (.test_skill.report_language //= (.default_language // "sk")) |
  (.test_skill.suggest_missing_acceptance_criteria //= true) |
  (.test_skill.test_runners //= {
    "php_unit": "auto",
    "php_browser": "auto",
    "js_unit": "auto",
    "js_browser": "auto"
  }) |
  (.test_skill.test_globs //= [
    "tests/**/*Test.php",
    "tests/**/*Spec.php",
    "tests/**/*.test.{ts,tsx,js,jsx,mjs,cjs}",
    "tests/**/*.spec.{ts,tsx,js,jsx,mjs,cjs}",
    "cypress/e2e/**/*.cy.{ts,tsx,js,jsx}",
    "tests/Browser/**/*.php",
    "tests/e2e/**/*.{ts,tsx,js}",
    "playwright/**/*.spec.{ts,tsx,js}",
    "e2e/**/*.{spec,test}.{ts,tsx,js}"
  ]) |
  (.test_skill.visual_file_patterns //= [
    "\\.(vue|jsx|tsx|svelte|blade\\.php|html)$",
    "^resources/views/"
  ]) |
  (.test_skill.max_browser_steps //= 25) |
  (.test_skill.report_output_dir //= "./.teamwork-task-test")
' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
chmod 600 "$CONFIG_FILE"
```

CLI flags override `.test_skill.*` values for the duration of this run.

---

## Step 3 — Fetch tasks via Teamwork REST API v3

Same auth pattern as `teamwork-task`:

```bash
TOKEN=$(jq -r '.teamwork.api_token' "$CONFIG_FILE")
BASE=$(jq  -r '.teamwork.base_url'  "$CONFIG_FILE")
AUTH="${TOKEN}:xxx"
```

**Single task** (`URL_KIND=task`):
```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${ENTITY_ID}.json"
```

**Tasklist** (`URL_KIND=tasklist`):
```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json"

curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?pageSize=100&page=1"
```

Paginate while `.meta.page.hasMore == true`. Extract per task:
- `.task.id` / `.tasks[]` with `{id, name, description, projectId, estimateMinutes, priority, status, dueAt, commentsCount, tags}`.

By default skip tasks with `status == "completed"` unless the user passed `--include-completed` (allow this flag implicitly — it is a sensible escape hatch for QA passes on already-finished work).

If `commentsCount > 0`, fetch the comments (Teamwork often holds the freshest acceptance criteria in the last comment):

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}/comments.json?pageSize=100&page=1&orderBy=postedAt&orderMode=asc"
```

If a fetch returns HTTP 401 → token invalid; re-prompt the user (re-run Step 2's first-run flow). 403/404 → report and stop.

---

## Step 4 — Parse acceptance criteria from the task description

WAME convention: the description is split on the first horizontal rule (`<hr>`, `<hr/>`, `<hr />`, or markdown `---` / `***` / `___` on its own line) into:
- **`acceptance_criteria`** (content above the HR) — the checklist of conditions to verify
- **`final_summary`** (content below the HR) — the authoritative goal / business outcome

Same split as the `teamwork-task` plugin:
```bash
SPLIT_REGEX='(<hr[[:space:]]*/?>|^[[:space:]]*(---|\*\*\*|___)[[:space:]]*$)'
```

### Step 4.1 — Extract individual criteria

From `acceptance_criteria`, extract individual bullet/checkbox items. Be liberal in what is recognised — Teamwork's WYSIWYG produces a wide variety of HTML.

Recognise these item shapes (HTML stripped to plain text first; lowercase the marker):
- HTML: `<li>…</li>` inside `<ul>` / `<ol>`
- Checkbox HTML: `<input type="checkbox" …>` followed by a label
- Markdown bullets: lines starting with `- `, `* `, `+ `
- Markdown numbered: `1.`, `1)`
- Markdown checkboxes: `- [ ]`, `- [x]`
- Plain numbered lines if there are no bullets at all: `1.` / `1)` at start of line

Save each extracted criterion as an object:
```json
{
  "id": "ac-1",
  "raw": "User môže otvoriť zoznam faktúr cez menu",
  "kind": "behavior" | "ui" | "data" | "api" | "performance" | "security" | "unknown",
  "initial_status": "open" | "checked",   // if the source already marked it [x]
  "status": "pending",                     // will be set in Step 6
  "evidence": [],
  "manual_steps": null,
  "suggested": false
}
```

`kind` is classified heuristically by keywords for routing in Step 6:
- `ui` — *"vidí", "zobrazí", "tlačidlo", "modálne okno", "sees", "displays", "button", "modal", "page"*; file diff would touch `.vue` / `.blade.php` / `.tsx` / etc.
- `data` — *"v databáze", "uloží", "in DB", "persists", "row", "column"*
- `api` — *"endpoint", "POST", "GET", "API", "JSON"*
- `performance` — *"do X sekúnd", "rýchlejšie ako", "performance", "load time"*
- `security` — *"autorizácia", "permission", "role", "auth", "denied"*
- `behavior` — anything else with a verb
- `unknown` — short fragments without enough signal

### Step 4.2 — When acceptance criteria are missing or empty

If `acceptance_criteria` is empty, or fewer than 2 individual criteria were extracted, **and** `config.test_skill.suggest_missing_acceptance_criteria == true` (default), build a draft set of acceptance criteria from:
1. The task title
2. The `final_summary` (below HR)
3. The freshest comment(s)
4. Any attached spec files (`.md`, `.txt`, `.pdf` titles)

Draft 3–6 criteria, each phrased as a verifiable behaviour. Mark each with `"suggested": true` and `"status": "proposed"`. Render them to the user via **AskUserQuestion**:

- **Use these as-is** — proceed to Step 5 with them
- **Edit them** — user pastes a corrected list
- **Skip this task** — record the task in the report as `📝 missing acceptance criteria — not tested`

Never silently invent criteria and treat them as authoritative — always mark them as `suggested` so the final report shows them under a separate "Proposed" status.

---

## Step 5 — Detect the test stack and existing tests

Run **once per session**, cache the result. Reuse the detection from `teamwork-task` but extend with browser-side detection.

```bash
HAS_LARAVEL=0; HAS_PEST=0; HAS_PHPUNIT=0; HAS_DUSK=0
HAS_VITEST=0; HAS_JEST=0; HAS_PLAYWRIGHT=0; HAS_CYPRESS=0; HAS_SELENIUM=0
HAS_BROWSER_MCP=0

if [ -f composer.json ]; then
  jq -e '.require."laravel/framework"      // .["require-dev"]."laravel/framework"'      composer.json >/dev/null 2>&1 && HAS_LARAVEL=1
  jq -e '.require."pestphp/pest"           // .["require-dev"]."pestphp/pest"'           composer.json >/dev/null 2>&1 && HAS_PEST=1
  jq -e '.require."phpunit/phpunit"        // .["require-dev"]."phpunit/phpunit"'        composer.json >/dev/null 2>&1 && HAS_PHPUNIT=1
  jq -e '.require."laravel/dusk"           // .["require-dev"]."laravel/dusk"'           composer.json >/dev/null 2>&1 && HAS_DUSK=1
fi

if [ -f package.json ]; then
  jq -e '.dependencies.vitest                       // .devDependencies.vitest'                       package.json >/dev/null 2>&1 && HAS_VITEST=1
  jq -e '.dependencies.jest                         // .devDependencies.jest'                         package.json >/dev/null 2>&1 && HAS_JEST=1
  jq -e '.dependencies["@playwright/test"]          // .devDependencies["@playwright/test"]'          package.json >/dev/null 2>&1 && HAS_PLAYWRIGHT=1
  jq -e '.dependencies.cypress                      // .devDependencies.cypress'                      package.json >/dev/null 2>&1 && HAS_CYPRESS=1
  jq -e '.dependencies["selenium-webdriver"]        // .devDependencies["selenium-webdriver"]'        package.json >/dev/null 2>&1 && HAS_SELENIUM=1
fi
```

Then detect whether the **chrome-devtools MCP** is available in this session by checking the system reminder / MCP list (the model can see the available tools — look for any `mcp__*chrome-devtools__*` tool). If present → `HAS_BROWSER_MCP=1`. This lets the skill drive a real browser without requiring the project to have Cypress / Playwright / Dusk installed.

Discover existing test files using the globs in `config.test_skill.test_globs`. Build a single index:

```
TEST_INDEX
├─ test_file: tests/Feature/InvoicePdfTest.php
│  test_names: ["it exports invoice as PDF", "it includes tenant theme"]
│  keywords:   [invoice, pdf, export, tenant, theme]
├─ test_file: cypress/e2e/login.cy.ts
│  test_names: ["user can log in", "shows error on wrong password"]
│  keywords:   [login, user, error, password]
…
```

Keyword extraction per test file:
- For PHP: grep `it\(`, `test\(`, `function test_…`, class names (`InvoicePdfTest` → `invoice`, `pdf`).
- For JS / TS: grep `describe(`, `it(`, `test(` calls; pull arguments (best-effort with `grep -oE`).
- Tokenise: lowercase, split on non-alphanumerics, drop stopwords (`should`, `does`, `the`, `a`, `to`, `it`, …), keep the rest as a keyword set.

This is heuristic — do **not** be precious about it. The goal is just to give the criterion-mapping step in Step 6 a decent shortlist of candidate tests for each criterion.

---

## Step 6 — Verify each acceptance criterion

For each task, for each acceptance criterion, attempt verification in this order. **Stop at the first method that produces a definitive result** (✅ or ❌) and record it; otherwise fall through to manual.

### 6.1 Map criterion → candidate tests

Tokenise the criterion's `raw` text the same way as test keywords (lowercase, drop stopwords). Score each test file in the `TEST_INDEX` by token overlap (Jaccard or just count). Take the top 3 candidates with score ≥ 1.

If `config.test_skill.run_tests == "never"` → skip execution, fall straight through to the visual / manual fallbacks.

### 6.2 Execute candidate tests

Pick the highest-scoring candidate. Decide the runner:

| Test file path                                  | Runner command (default — override via `test_runners.*` config)        |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| `tests/Feature/…` / `tests/Unit/…` (Pest)       | `./vendor/bin/pest --filter "<test name>"`                              |
| `tests/Feature/…` / `tests/Unit/…` (PHPUnit)    | `./vendor/bin/phpunit --filter "<test name>"`                           |
| `tests/Browser/…` (Laravel Dusk)                | `php artisan dusk --filter "<test name>"`                               |
| `cypress/e2e/…`                                 | `npx cypress run --spec "<file>" --headless`                            |
| `tests/e2e/…` / `playwright/…` (Playwright)     | `npx playwright test "<file>" -g "<test name>"`                         |
| `tests/**/*.{test,spec}.{ts,js,tsx,jsx}` (Vitest) | `npx vitest run -t "<test name>" "<file>"`                              |
| Same path family but project uses Jest          | `npx jest --testNamePattern "<test name>" "<file>"`                     |

Choose Pest over PHPUnit if `HAS_PEST=1`. Choose Vitest over Jest if both are present. If `test_runners.<lang>_*` overrides the default, honour that.

Capture the command's exit code and stdout. Cap stdout at the last 200 lines so the report does not explode.

**Verdict mapping:**
- Exit 0 → criterion status `✅ verified` with evidence `{ test_file, test_name, runner, duration_ms }`.
- Exit non-zero with a clear assertion failure in the output → criterion status `❌ failed` with the failure message (trim to the assertion line + 5 lines of context).
- Exit non-zero with infrastructure error (no DB, missing env, exit 255 from a runner crash) → do not call this a failure; fall through to 6.3 / 6.4 and record this in the criterion's `notes` ("test runner errored — not a real failure").

### 6.3 Visual verification via chrome-devtools MCP

If the criterion is `kind == "ui"` (or `unknown` and the task touches files matching `config.test_skill.visual_file_patterns`), and `config.test_skill.visual_mode != "skip"`, and `HAS_BROWSER_MCP == 1`:

1. Determine a URL to test. Sources in priority order:
   - `APP_URL` from `.env` or `.env.testing`
   - `process.env.APP_URL` in `package.json` scripts
   - Default `http://localhost:8000` for Laravel, `http://localhost:5173` for Vite, `http://localhost:3000` for Next/Nuxt — print a one-line warning that we assumed this.
2. Confirm the dev server is up: `curl -sS -o /dev/null -w "%{http_code}" "$APP_URL"`.
   - If it is not up, do **not** auto-start anything in the project (could collide with the user's running server). Mark this criterion `📋 manual — dev server not running` and continue.
3. Use the chrome-devtools MCP to:
   - `navigate_page` to the relevant route (derive from criterion text — e.g. "faktúry" → try `/invoices`, `/faktury`, `/nova/resources/invoices`; if unsure, navigate to the app root and use `take_snapshot` to find a matching link).
   - `take_screenshot` and save under `${REPORT_DIR}/screenshots/<task-id>/<ac-id>.png`.
   - For each verifiable noun in the criterion (a heading, a button label, a row of data), use `take_snapshot` and assert the text is present.
   - Cap to `config.test_skill.max_browser_steps` interactions per criterion (default 25) to avoid runaway sessions.
4. Result:
   - All required assertions present → `✅ verified` with evidence `{ method: "chrome-devtools", url, screenshot, asserted_text }`.
   - Required text missing → `❌ failed` with the diff.
   - Could not reach the page (404, 500, auth wall) → fall through to 6.4.

This is best-effort. The MCP-based check is intentionally framed as evidence, not gospel — the user reviews the screenshot in the final report.

### 6.4 Write a manual test scenario

If 6.1–6.3 could not produce a definitive verdict, write a concrete manual scenario for the criterion. Render it as a numbered checklist a human can follow without re-reading the task:

```
📋 Manual scenario for AC-3 "Email sa odošle pri uložení"
1. Otvor /invoices/create v prehliadači.
2. Vyplň povinné polia (klient, dátum, položka).
3. Stlač "Uložiť".
4. Skontroluj mailtrap.io / log inbox — má prísť e-mail na adresu klienta.
5. ✅ Pass if e-mail bol odoslaný do 30 sekúnd.
6. ❌ Fail if e-mail neprišiel alebo prišiel bez prílohy s PDF.
```

Set status `📋 manual` with the scenario stored on the criterion. Manual is a **valid** outcome — many criteria simply cannot be automated cheaply, and a clear manual recipe is the next best thing.

### 6.5 Skipped or proposed criteria

- Criteria where `suggested == true` and the user did not approve them in Step 4.2 → status `📝 proposed`.
- Criteria where the user explicitly skipped (via per-task confirmation if `comment_on_task` was enabled, or `--skip-ac=ac-2,ac-5`) → status `⏭️ skipped`.

---

## Step 7 — Per-task report

After all criteria for a task have been verified, render the task block:

```markdown
### [#<task-id>] <task title>
- **Stav testovania:** ✅ PASSED | ⚠️ PARTIAL | ❌ FAILED | 📋 MANUAL ONLY | 📝 MISSING AC
- **Goal:** <one-line from final_summary>
- **Test stack used:** Pest 2.x, Laravel Dusk, chrome-devtools MCP
- **Tests executed:** 7 (5 ✅, 1 ❌, 1 ⚠️ infra error)
- **Browser checks:** 2 screenshots saved to `.teamwork-task-test/screenshots/<task-id>/`

#### Akceptačné kritériá

| #  | Kritérium                                            | Stav         | Spôsob overenia                                                                |
|----|------------------------------------------------------|--------------|--------------------------------------------------------------------------------|
| 1  | User môže otvoriť faktúru cez menu                   | ✅ verified  | Pest — `tests/Feature/InvoiceListTest.php:42` (`it shows invoice list`)         |
| 2  | PDF má hlavičku firmy v ľavom hornom rohu            | ⚠️ partial   | Dusk render OK, font check manual — viď scenár AC-2                            |
| 3  | E-mail sa odošle klientovi pri uložení faktúry       | ❌ failed    | Pest — `tests/Feature/InvoiceMailTest.php:21` (assertion: mail count was 0)    |
| 4  | API endpoint vráti 422 pri chýbajúcom IBAN           | ✅ verified  | Pest — `tests/Feature/InvoiceApiTest.php:88`                                   |
| 5  | (navrhnuté) Validácia formátu IBAN podľa krajiny     | 📝 proposed  | Chýba v zadaní — odporúčam doplniť                                             |
| 6  | Akcia "Stornovať" je viditeľná len pre rolu Manager  | 📋 manual    | viď scenár nižšie                                                              |

**Status legend:** ✅ verified by test ・ ⚠️ partial (some evidence) ・ ❌ failed ・ 📋 manual (scenario provided) ・ 📝 proposed (AC missing) ・ ⏭️ skipped

##### Detail per AC

**AC-2** — `tests/Browser/InvoicePdfTest.php` ran in 4.2 s, screenshot `.teamwork-task-test/screenshots/123456/ac-2.png`. Font family in the rendered header could not be asserted by the Dusk runner — needs human eyeball on the screenshot.

**AC-3** — Pest output:
```
FAIL  tests/Feature/InvoiceMailTest.php
✗ it sends an email to the customer after saving
  Mail::assertSent(InvoiceCreated::class) → expected 1, got 0
  at tests/Feature/InvoiceMailTest.php:21
```

**AC-6** — Manuálny scenár:
1. Prihlás sa ako používateľ s rolou **Manager** → `/login`
2. Otvor `/invoices/<existing-id>`
3. Skontroluj že v hornej lište je tlačidlo **Stornovať**
4. Odhlás sa, prihlás sa ako rola **Accountant**, otvor tú istú stránku
5. ✅ Pass ak tlačidlo **NIE JE** viditeľné pre Accountanta
6. ❌ Fail ak je viditeľné pre Accountanta alebo ak nie je viditeľné pre Manager
```

The bulk of the value of this skill is in this per-task block — every acceptance criterion has a row in the table, every row has a status, every row points at the evidence. The user can scan the table at a glance, then dive into the per-AC detail for anything that is not ✅.

---

## Step 8 — Tasklist summary (when `URL_KIND=tasklist`)

After all tasks have been processed, render a summary table for the whole list:

```markdown
## Tasklist summary — "<tasklist name>"

| # | Task ID | Title                              | Status      | AC: ✅ / ⚠️ / ❌ / 📋 / 📝 | Tests | Time |
|---|---------|------------------------------------|-------------|--------------------------|-------|------|
| 1 | 44740914| Annual advance PDF restructure     | ✅ PASSED   | 6 / 0 / 0 / 1 / 0        | 8 ✅  | 15m  |
| 2 | 44740916| Cent precision in Ročne column     | ⚠️ PARTIAL  | 3 / 1 / 0 / 1 / 1        | 4 ✅ 1 ⚠️ | 20m  |
| 3 | 44740918| Indent MESAČNÝ heading             | 📋 MANUAL   | 0 / 0 / 0 / 2 / 0        | —     | 5m   |

**Totals**
- Acceptance criteria: 12 ✅ verified ・ 1 ⚠️ partial ・ 0 ❌ failed ・ 4 📋 manual ・ 1 📝 proposed
- Tests executed: 13 ✅ pass ・ 0 ❌ fail ・ 1 ⚠️ infra error
- Time logged to Teamwork: 40 min (if `--time-log=true`)
- Comments posted to Teamwork: 0 (default) / 3 (if `--comment-on-task=true`)
```

Followed by a short legend identical to the per-task one. Print this as the **last** thing in the run.

---

## Step 9 — Optional write-backs to Teamwork

By default this skill is **read-only** against Teamwork. Two opt-in write-backs:

### 9.1 Time log

If `config.test_skill.time_log == true` (default) **and** the user did not pass `--time-log=false`, log a single time entry per task with the elapsed QA time. Reuse the **same sequential, non-overlapping cursor** logic as the `teamwork-task` plugin (Step 5.5 / 6.8 there) — the cursor lives in `SESSION_CURSOR_TS` and advances by exactly the logged duration after each successful POST. This is critical: a user who runs `/teamwork-task` and `/teamwork-task-test` back-to-back must not get overlapping timesheet rows.

> **TIMEZONE CONTRACT (critical).** Teamwork's time API is asymmetric: `GET …/time.json`
> returns `timeLogged` in **UTC** (trailing `Z`), but the **POST/PATCH `time` field is
> interpreted in the user's LOCAL/profile timezone**. When you reuse the `teamwork-task`
> cursor logic, parse `timeLogged` as UTC (`date -ju -f "…Z"` — the `-u` is mandatory on
> macOS, the trailing `Z` is a literal, not a zone directive) but format the POST `time`
> in LOCAL (`date -r "$TS" +%H:%M:%S`, **no** `-u`). Mixing them up shifts every QA entry
> by the local offset, so logs land hours early and overlap the implementation entries.
> If you ever need to correct a misplaced entry, `PATCH …/projects/api/v3/time/{id}.json`
> with `{"timelog":{"time":"HH:MM:SS"}}` (local time).

Description tone: business, in `config.test_skill.report_language` (default `sk`). Examples (Slovak):
- *"Otestované akceptačné kritériá pre modul Faktúry — 6/7 splnených, 1 manuálny scenár."*
- *"QA pas pre Stornovať akciu — všetky AC overené Pestom."*

Endpoint: `POST ${BASE}/projects/api/v3/tasks/${TASK_ID}/time.json` — same payload shape as the `teamwork-task` plugin.

### 9.2 Comment on the task

If `config.test_skill.comment_on_task == true` (or `--comment-on-task=true`), post the per-task report (Step 7) as a comment on the Teamwork task. Strip the screenshots from the markdown (Teamwork rendering does not show local file paths) and replace them with a plain mention: *"Screenshot uložený lokálne pri spustení."*

Endpoint: `POST ${BASE}/projects/api/v3/tasks/${TASK_ID}/comments.json` with payload:
```json
{ "comment": { "body": "<markdown body>", "contentType": "MARKDOWN" } }
```

Some Teamwork tenants expect `"HTML"` content type instead. If MARKDOWN returns 4xx, retry once with HTML (do a minimal markdown → HTML conversion: `**` → `<strong>`, line breaks → `<br>`, tables stay as markdown which most tenants accept inline).

### 9.3 Board moves (optional, off by default)

If `config.test_skill.move_on_pass` is set to a non-empty stage name **and** every AC for the task ended ✅, post the task to that stage (reuse the workflow stage resolution from the `teamwork-task` plugin Step 3.3). Same for `move_on_fail` if any AC ended ❌. By default both are empty strings → the skill never touches the board.

---

## Blocker handling

Stop and ask via **AskUserQuestion** only on these genuine blockers:
- URL cannot be parsed.
- API returns 401 (token invalid / expired).
- The task has no acceptance criteria *and* `suggest_missing_acceptance_criteria == false` — ask whether to skip the task or enable suggestion.
- The dev server is needed for visual verification but is not running — ask whether to switch to manual scenarios or to skip the criterion.
- The chosen test runner is not installed in the project (e.g. Cypress imports but `npx cypress` is missing) — ask whether to skip or install.

Everything else degrades silently with a single-line warning to stderr and continues:
- Comment / time log POST failures.
- Single test execution failures (an `❌ failed` AC is a *result*, not a blocker).
- Browser MCP transient errors — record the criterion as `📋 manual` and move on.
- Attachment download failures.

---

## Output & file artefacts

Aside from the inline report printed to stdout, the skill writes:

```
${REPORT_DIR}/                              # default `./.teamwork-task-test/`
├── <task-id>/
│   ├── report.md                           # the per-task block (same as Step 7)
│   ├── ac.json                             # machine-readable AC + results
│   └── screenshots/
│       ├── ac-2.png
│       └── ac-6.png
└── summary.md                              # the Step 8 tasklist summary
```

`ac.json` shape:
```json
{
  "task_id": 123456,
  "title": "…",
  "status": "partial",
  "criteria": [
    {
      "id": "ac-1",
      "raw": "User môže otvoriť faktúru",
      "status": "verified",
      "evidence": [{ "method": "pest", "file": "tests/Feature/InvoiceListTest.php", "test_name": "it shows invoice list", "duration_ms": 480 }]
    },
    …
  ]
}
```

This makes the report grep-friendly and lets CI / other tooling pick it up.

Add `${REPORT_DIR}` to `.gitignore` automatically using the same idempotent
pattern as the `teamwork-task` plugin (`/.teamwork-task-test/`).

---

## Security

- The API token lives only in `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (chmod 600).
- Never `echo` the token, never paste it into the report, never write it to `ac.json` / `report.md`.
- Pass auth to `curl` via `-u "$TOKEN:xxx"`, never via the URL query string.
- Browser screenshots may contain confidential customer data — they live under the auto-`.gitignore`d `${REPORT_DIR}` so they cannot accidentally be committed.
