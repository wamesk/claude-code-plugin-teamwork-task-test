# Teamwork Task Test — Claude Code Plugin

> Verify a Teamwork.com task or whole tasklist against your codebase — every
> acceptance criterion is individually checked, automated where possible,
> and turned into a manual scenario where not. The output is a single report
> where every AC is annotated with ✅ verified / ⚠️ partial / ❌ failed /
> 📋 manual / 📝 proposed.

This is the **read-only sibling** of [`teamwork-task`](https://github.com/wamesk/claude-code-plugin-teamwork-task).
`teamwork-task` *implements* tickets. `teamwork-task-test` *checks whether
they were actually delivered.* The two plugins share the same Teamwork API
token config — set it up once, both skills use it.

## Installation

```text
/plugin marketplace add wamesk/claude-code
/plugin install teamwork-task-test@wame
```

(If you already added the WAME marketplace, only the second line is needed.)

## Quickstart

```text
/teamwork-task-test https://wame.teamwork.com/app/tasks/44740914
```

or for a whole tasklist:

```text
/teamwork-task-test https://wame.teamwork.com/app/tasklists/3335435
```

On the first run, you'll be asked once for your Teamwork base URL and API
token (paste it from `https://<workspace>.teamwork.com/launchpad/apikey/manage`).
The token is stored at `~/.claude/plugins/data/teamwork-task-wamesk/config.json`
with `chmod 600` and survives plugin updates. **If you've already configured
the `teamwork-task` plugin, nothing extra is needed** — the two skills read
the same file.

## What it does, step by step

1. **Parses the URL** — single task or tasklist, any of Teamwork's common URL shapes.
2. **Loads the shared config**, prompts for a token only on first run.
3. **Fetches the task(s)** via Teamwork REST API v3. Pulls comments when the
   task has any — the freshest AC often live in the last comment.
4. **Parses the description** on the first horizontal rule into
   `acceptance_criteria` (above) and `final_summary` (below). Extracts each
   AC as an individual item (HTML `<li>`, markdown bullets, checkboxes, …).
5. **If acceptance criteria are missing or thin**, drafts 3–6 from the title
   + final summary + freshest comments, marks them as `suggested`, and asks
   you to approve / edit / skip before testing.
6. **Detects the test stack** — Pest, PHPUnit, Laravel Dusk, Cypress,
   Playwright, Vitest, Jest, Selenium WebDriver — and whether the
   `chrome-devtools` MCP is available in the session.
7. **Indexes existing tests** under `tests/`, `cypress/e2e/`, `playwright/`,
   `tests/Browser/`, `tests/e2e/` … and extracts keywords from
   `it()` / `test()` / `describe()` names.
8. **For each acceptance criterion**, tries verification in order:
   1. Map → score → run the best-matching existing test with the right
      runner. ✅ on green exit, ❌ on a clean assertion failure.
   2. If the criterion is UI-shaped and the chrome-devtools MCP is around,
      navigate to a derived URL, take a screenshot, assert visible text.
   3. Otherwise, write a concrete numbered **manual scenario** with explicit
      pass / fail criteria.
9. **Puts every green test through a negative control** *(1.1.0)* — applies the
   smallest possible revert of the fix, confirms the test goes red, restores from
   a backup kept outside the repo. A test that passes with *and* without the fix
   gets its criterion downgraded to ⚠️ partial, because it proves nothing.
10. **Reviews four cross-cutting dimensions** over the task's own diff *(1.1.0)*
    — **UI/UX & accessibility**, **performance**, **security**,
    **reachability** — and reports those findings separately from the AC table.
    This is the half of QA that acceptance criteria never ask for.
11. **Renders a per-task report** with a status table for every AC and the
    evidence linked next to it (test file:line, runner, screenshot path, or
    manual scenario reference), plus the findings and negative-control tables.
12. **Tasklist run** also produces a final summary table over all tasks.
13. **Ticks the met criteria** to `- [x]` in the Teamwork description *(1.1.0,
    default ON)* under a byte-exact safety contract, **logs time** (default ON,
    sequential and non-overlapping like the `teamwork-task` plugin) and
    **optionally posts the report as a comment** (default OFF).

## The four review dimensions *(1.1.0)*

Acceptance criteria describe what the author thought to ask for. They are quiet
about the greyed-out button that gives no reason, the page nobody can reach, the
query that runs once per row, and the crafted request that answers 500. So every
run also reviews the task's diff along four axes and reports what it finds with a
severity and a file:line — never as a vague worry.

| Dimension | What it actually checks |
|---|---|
| **UI/UX & a11y** | A disabled control carries `aria-disabled` **and** a reason — greying a button out tells a sighted user it does not work and tells a screen reader nothing. Icon-only controls have accessible names. Every new translation key resolves to text rather than to itself (the silent miss where a framework walks a dotted key, finds a string at the prefix, and renders the raw key). Labels, `alt`, contrast, layout stability. |
| **Performance** | Real LCP / CLS / INP / TTFB against configurable budgets, plus the diff read for what only hurts at scale: N+1, an unchunked batch, a table walked in PHP where a `WHERE` would do. Every finding names the row count at which it starts to matter, and says whether this change caused it or walked past it. |
| **Security** | Authorization and tenant isolation on new routes, object-scoped checks vs. role-only checks (IDOR), mass assignment, injection through author-controlled strings, secrets, and unhandled error paths on reachable routes. Notes that with `APP_DEBUG=false` a well-worded exception reaches the user as a generic 500 — so a named exception helps the log, not the screen. |
| **Reachability** | Is every registered screen reachable **by clicking**? Screens are enumerated from the filesystem, menu links are read from the live rendered navigation, and screens reachable through a parent's relation tab are subtracted. What is left is findable only by typing the URL — which means findable by nobody. `allow_orphans` covers deliberate URL-only pages. |

Scope them per run with `--dimensions=ui_ux,security` or switch them off with
`--dimensions=none`. A dimension that could not run is named in the report,
because a silently skipped dimension reads as a clean bill of health.

## Example output

```markdown
### [#44740916] Cent precision in Ročne column
- **Stav testovania:** ⚠️ PARTIAL
- **Test stack used:** Pest 2.x + Laravel Dusk + chrome-devtools MCP
- **Tests executed:** 4 (3 ✅, 0 ❌, 1 ⚠️ infra error)

| #  | Kritérium                                       | Stav         | Spôsob overenia                                                |
|----|-------------------------------------------------|--------------|----------------------------------------------------------------|
| 1  | Ročne column zachová desatinné centy            | ✅ verified  | Pest — `tests/Feature/InvoiceAnnualTest.php:42`                |
| 2  | Header zobrazí dátumy pod titulkom              | ⚠️ partial   | Dusk render OK, font check manual — viď scenár AC-2            |
| 3  | API vráti rovnaké hodnoty ako PDF               | ✅ verified  | Pest — `tests/Feature/InvoiceApiAnnualTest.php:88`             |
| 4  | (navrhnuté) Validácia záporných centov          | 📝 proposed  | Chýba v zadaní — odporúčam doplniť                             |

#### Nálezy mimo akceptačných kritérií

| Rozmer | Sev | Nález | Kde | V tomto diffe? |
|--------|-----|-------|-----|----------------|
| UI/UX | 🟠 medium | Neaktívne tlačidlo nehovorí prečo — `<span>` bez `aria-disabled` aj bez `title` | `resources/views/nova/invoice/buttons-card.blade.php:41` | áno → opravené |
| Reachability | 🟡 low | `Country` je registrovaná obrazovka, nevedie na ňu odkaz ani relačný tab | `app/Nova/Country.php` | nie — staršie |

#### Negatívne kontroly

| Kritérium | Čo som vrátil | Test spadol? |
|-----------|---------------|--------------|
| AC-1 | `?? ''` zo šablóny | ✅ áno, 3 testy na `ViewException` |
```

## Configuration

The config file lives at:

```text
~/.claude/plugins/data/teamwork-task-wamesk/config.json
```

It is **shared with the `teamwork-task` plugin**. This plugin only adds the
`test_skill` subtree:

```jsonc
{
  "teamwork": {
    "base_url": "https://acme.teamwork.com",
    "api_token": "***"
  },
  "default_language": "sk",

  "test_skill": {
    "run_tests": "auto",                 // auto | never | always
    "visual_mode": "auto",               // auto | browser | skip
    "comment_on_task": false,            // post the report as a comment
    "time_log": true,                    // QA work appears in the timesheet
    "move_on_pass": "",                  // e.g. "Testing" to move on full ✅
    "move_on_fail": "",                  // e.g. "In progress" to move on ❌
    "report_language": "sk",             // sk | en
    "suggest_missing_acceptance_criteria": true,

    "test_runners": {
      "php_unit": "auto",                // auto | pest | phpunit
      "php_browser": "auto",             // auto | dusk | selenium
      "js_unit": "auto",                 // auto | vitest | jest
      "js_browser": "auto"               // auto | playwright | cypress
    },

    "test_globs": [ "tests/**/*Test.php", "cypress/e2e/**/*.cy.{ts,tsx,js,jsx}", "…" ],
    "visual_file_patterns": [ "\\.(vue|jsx|tsx|svelte|blade\\.php|html)$", "^resources/views/" ],
    "max_browser_steps": 25,
    "report_output_dir": "./.teamwork-task-test",

    "review_dimensions": ["ui_ux", "performance", "security", "reachability"],
    "negative_control": true,
    "tick_acceptance_criteria": true,
    "performance_budgets": { "lcp_ms": 2500, "cls": 0.1, "inp_ms": 200, "ttfb_ms": 800 },
    "reachability": { "enabled": true, "screen_globs": [ "app/Nova/**/*.php", "…" ], "allow_orphans": [] },
    "a11y_checks": [ "disabled_control_has_reason", "translation_key_resolves_to_text", "…" ]
  }
}
```

All `test_skill.*` keys can be temporarily overridden via CLI flags — see
`SKILL.md` for the full list.

## CLI flags

| Flag | Values | Default | Notes |
|------|--------|---------|-------|
| `--run-tests=` | `auto`, `never`, `always` | `auto` | Whether to actually execute discovered tests. |
| `--visual=` | `auto`, `browser`, `skip` | `auto` | Use the chrome-devtools MCP for UI-shaped AC. |
| `--dimensions=` | subset of `ui_ux,performance,security,reachability`, or `none` | all four | Which cross-cutting review dimensions to run. |
| `--negative-control=` | `true`, `false` | `true` | Revert each fix and confirm its test goes red before trusting it. |
| `--tick-ac=` | `true`, `false` | `true` | Tick met criteria to `- [x]` in the Teamwork description. |
| `--comment-on-task=` | `true`, `false` | `false` | Post the per-task report as a Teamwork comment. |
| `--time-log=` | `true`, `false` | `true` | Log QA time back to Teamwork (sequential, no overlaps). |
| `--language=` | `sk`, `en` | from config (`sk`) | Language for the report and AC suggestions. |
| `--include-completed` | (flag) | off | Don't skip tasks already marked `completed` in Teamwork. |
| `--skip-ac=` | comma list of AC IDs (`ac-2,ac-5`) | — | Skip specific criteria for this run. |

## Output files

```text
./.teamwork-task-test/                  # under report_output_dir, auto-.gitignored
├── <task-id>/
│   ├── report.md                       # human-readable per-task block
│   ├── ac.json                         # machine-readable AC + verdicts
│   └── screenshots/
│       ├── ac-2.png
│       └── ac-6.png
└── summary.md                          # tasklist roll-up
```

`ac.json` is grep-friendly and intended for CI integrations. Since 1.1.0 it also
carries `review_findings`, `dimensions_run` and `dimensions_skipped` — the last
two are what make an empty findings array meaningful, because without them "no
findings" and "nobody looked" read the same.

## What this plugin will **never** do

- Move the Teamwork card to a different column (unless you explicitly set
  `move_on_pass` / `move_on_fail` in the config).
- Mark the task as `completed` in Teamwork.
- Edit or delete a Teamwork comment that already exists.
- Rewrite any part of the Teamwork description other than an acceptance-criteria
  checkbox. The tick in 1.1.0 is a character-level substitution guarded by
  assertions, and it aborts rather than sending a body it cannot prove is
  minimal — an inline image in a Teamwork description exists *only* in that
  markdown, so a careless rewrite would delete a screenshot for good.
- Tick a criterion that was not proven. Only ✅ is ticked; ⚠️ partial, 📋 manual
  and ❌ failed stay open.
- Push to a git remote.
- Leave your code changed. It **does** edit code during a negative control
  (Step 6.5.5) — that is the point: revert the fix, watch the test fail — but it
  backs the file up outside the repo first, restores it, and verifies the restore
  with `git status`. A dirty tree at the end is treated as a blocker, not a
  warning. Beyond that it does not implement anything: this is a verification
  skill — for implementation, use
  [`teamwork-task`](https://github.com/wamesk/claude-code-plugin-teamwork-task).
- Echo the API token to stdout, into the report files, into the commit
  history, or anywhere it could leak.

## Relationship to other plugins

- **`teamwork-task`** — implements tasks (writes code, commits, logs time,
  moves the board). Shares the same config and token. Run it first.
- **`teamwork-task-test`** (this plugin) — verifies the result. Run it
  after, on the same task or tasklist URL, to get a per-criterion QA report.
- **`teamwork-tasks-from-dnr`** — turns a *Detailný návrh riešenia*
  document into a Teamwork-import-ready tasklist. Run it before any of the
  above, to populate Teamwork in the first place.

## License

MIT — see [`LICENSE`](./LICENSE).
