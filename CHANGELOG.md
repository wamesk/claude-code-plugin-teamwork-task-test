# Changelog

All notable changes to this plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.2] — 2026-06-11

### Fixed

- **URL parser broken on macOS/BSD sed (Step 1).** `ENTITY_ID` and `URL_KIND`
  used `|` as both the `s` delimiter and the `(tasks|tasklists)` alternation,
  so BSD/macOS `sed` aborted with `RE error: parentheses not balanced` and both
  values came back empty for every URL — breaking the whole skill on macOS.
  Switched the `s` delimiter to `#` on both lines (regex unchanged); a `tasks`
  URL now yields the numeric id and `task` as expected.
- **Shell injection via grepped test names (Step 6.2).** Test names extracted
  from test files were interpolated into double-quoted runner arguments
  (`--filter "…"`, `-g "…"`, `-t "…"`, `--testNamePattern "…"`). A name
  containing a double quote plus `$(…)` or backticks broke out of the quoting
  and executed arbitrary shell. The name (and the file path) are now passed as
  `printf %q`-escaped, single-quoted-safe literals.
- **`jq` aborting on Teamwork control characters (Steps 3–4).** Raw curl JSON
  was piped straight into `jq`, which dies on Teamwork's unescaped control
  characters (`control characters U+0000–U+001F must be escaped`) — a single
  pasted control byte killed the run. Documented a tolerant sanitize step
  (`python3 json.loads(strict=False)` re-emit, or pre-stripping the control
  bytes with `tr`) applied to every Teamwork response before parsing.

### Changed

- Removed the `version:` field from the SKILL.md YAML frontmatter — `plugin.json`
  is now the single source of truth for the plugin version.

## [1.0.1] — 2026-06-10

### Fixed

- Documented the **TIMEZONE CONTRACT** for QA time logs in Step 9.1. The cursor
  logic reused from `teamwork-task` must parse Teamwork's UTC `timeLogged` with
  `date -ju` (the `-u` is mandatory on macOS — the trailing `Z` is a literal,
  not a zone directive) but format the POST `time` field in **local** time
  (`date -r`, no `-u`), because Teamwork interprets the posted `time` in the
  user's local/profile timezone. Mixing the two shifted QA entries by the local
  offset, landing them hours early and overlapping the implementation entries.
  Includes the `PATCH …/time/{id}.json` recipe to correct a misplaced entry.

## [1.0.0] — 2026-05-27

### Added

- Initial release of `/teamwork-task-test`.
- Fetch a Teamwork.com **single task** or **whole tasklist** by URL via the
  REST API v3 (HTTP Basic auth, `<api-token>:xxx`). Comments are pulled
  automatically when `commentsCount > 0` because the freshest acceptance
  criteria often live in the last comment.
- **HR-split** parser for task descriptions — content above the first
  horizontal rule (`<hr>` or markdown `---` / `***` / `___`) becomes
  `acceptance_criteria`; content below becomes the `final_summary`.
- **Per-criterion** extraction from `acceptance_criteria` — supports HTML
  `<li>`, HTML checkboxes, markdown bullets (`-`, `*`, `+`), markdown numbered
  lists, and markdown checkboxes (`- [ ]`, `- [x]`).
- **Missing-criteria mode** — when no AC are present, the skill drafts 3–6
  verifiable criteria from the task title + `final_summary` + freshest
  comments, marks them as `suggested`, and asks the user to approve / edit /
  skip via `AskUserQuestion` before testing.
- **Test stack detection** — Pest, PHPUnit, Laravel Dusk, Cypress, Playwright,
  Vitest, Jest, Selenium WebDriver. Cached once per session.
- **Chrome DevTools MCP** integration — when the MCP is available in the
  session and the criterion is UI-shaped, the skill drives a real browser:
  navigates to a derived URL, takes a screenshot, asserts visible text, and
  records the evidence on the criterion. Capped at
  `test_skill.max_browser_steps` interactions per criterion (default 25).
- **Test mapping** — per criterion, the skill scores existing test files by
  keyword overlap and runs the top candidate with the appropriate runner
  (Pest → `vendor/bin/pest --filter`, Cypress → `cypress run --spec`, etc.).
  Stops at the first definitive ✅ / ❌ verdict.
- **Manual scenario writer** — when no automated path can produce a verdict,
  the skill writes a numbered, human-followable scenario with explicit
  pass / fail criteria. Manual is treated as a *valid* outcome.
- **Per-task report** — every acceptance criterion gets its own row in a
  status table (✅ verified / ⚠️ partial / ❌ failed / 📋 manual / 📝
  proposed / ⏭️ skipped), with the evidence (test file:line, runner,
  screenshot path, or manual scenario reference) linked directly.
- **Tasklist summary** — at the end of a tasklist run, a single roll-up table
  with per-task status, AC tally, test count, and time logged.
- **Read-only by default** — does not move the board card, does not complete
  the task, does not post a comment. Time logging is the only write-back that
  is on by default (because QA is real work) and can be disabled with
  `--time-log=false`.
- **Sequential time logs** — reuses the same `SESSION_CURSOR_TS` logic as the
  `teamwork-task` plugin so a `/teamwork-task` run immediately followed by
  `/teamwork-task-test` produces strictly contiguous timesheet rows with no
  gaps and no overlaps.
- **Shared config** with the `teamwork-task` plugin
  (`~/.claude/plugins/data/teamwork-task-wamesk/config.json`) — the API token
  is configured once and used by both skills. New keys live under
  `.test_skill.*` and are added by an idempotent `jq` migration on first run.
- **Optional write-backs** — `--comment-on-task=true` posts the report as a
  Teamwork comment (with markdown / HTML retry); `move_on_pass` /
  `move_on_fail` in the config can move the card after a verdict.
- **Machine-readable output** — `${REPORT_DIR}/<task-id>/ac.json` mirrors the
  per-task report so CI and downstream tooling can pick it up.
- Auto-`.gitignore` for `${REPORT_DIR}` (`./.teamwork-task-test/` by default).

### Security

- API token never echoed to stdout / commit messages / report files; passed to
  `curl` via `-u` (kept out of `ps`).
- Config file `chmod 600` on creation; `.gitignore` excludes `config.json`.
