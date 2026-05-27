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
9. **Renders a per-task report** with a status table for every AC and the
   evidence linked next to it (test file:line, runner, screenshot path, or
   manual scenario reference).
10. **Tasklist run** also produces a final summary table over all tasks.
11. **Optionally logs time** to Teamwork (default ON, sequential and
    non-overlapping like the `teamwork-task` plugin) and **optionally posts
    the report as a comment** (default OFF).

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
    "report_output_dir": "./.teamwork-task-test"
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

`ac.json` is grep-friendly and intended for CI integrations.

## What this plugin will **never** do

- Move the Teamwork card to a different column (unless you explicitly set
  `move_on_pass` / `move_on_fail` in the config).
- Mark the task as `completed` in Teamwork.
- Edit or delete a Teamwork comment that already exists.
- Push to a git remote.
- Modify your code. This is a verification skill, not an implementation one
  — for implementation, use [`teamwork-task`](https://github.com/wamesk/claude-code-plugin-teamwork-task).
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
