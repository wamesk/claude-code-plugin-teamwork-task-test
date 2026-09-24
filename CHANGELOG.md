# Changelog

All notable changes to this plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] — 2026-09-24

One advisory review dimension and a round of fixes. The fixes come first in
importance: the count gate, the invalid `orderBy`, the missing `postedAt` field
and JSON piped through `echo` are four independent defects, each of which let a
QA run carry on with **no comments** and no error at all. They were reported in
a colleague's "Štyri cesty k nule" analysis and re-verified with read-only GETs
against a completed task that has 8 comments. The remaining fixes are the
`projectId` lookup and the zsh sweep that the same analysis prompted, plus the
acceptance-criteria block lookup the sibling plugins' new cross-cutting
criteria depend on.

### Added

- **`framework` — an advisory fifth review dimension (new Step 6.6.5).** The
  build-side plugins now use the idioms and built-in features of the framework
  versions a project actually has installed, instead of patterns remembered from
  older versions. The QA pass reads the task's diff for the same thing, but a
  tester must not modernize working code, so this dimension only
  **recommends**:
  - It never edits a file, never applies a recommendation, never downgrades or
    fails a criterion, never blocks, and never counts toward `Stav testovania`,
    the tasklist totals or any pass / fail roll-up. In `ac.json` every item
    carries `"dimension": "framework"`, `"severity": "info"`, `"fixed": false`
    and `"advisory": true`, and a CI gate must skip `advisory: true` items.
  - The versions come from the lock files (`composer.lock`, else
    `vendor/composer/installed.json`; `node_modules`, else `package-lock.json`;
    `.browserslistrc` / `browserslist`, `.nvmrc` / `engines`) or from Laravel
    Boost's `application-info` — never from memory; a bare manifest constraint
    is printed as one and treated as a floor — and are recorded in
    `ac.json` as `framework_versions`. The idiom is checked in current docs:
    Boost `search-docs`, else the context7 MCP, else the official docs. With no
    docs source reachable the dimension issues no recommendations and says so,
    because a recommendation from stale model memory is exactly what it exists
    to avoid.
  - Each recommendation points at a line of the task's own diff and names the
    installed version, the concrete replacement that version offers, and the
    docs page checked. *"Could be more modern"* without such a replacement is
    not a recommendation, and neither is anything outside the diff or anything
    that contradicts the project's `CLAUDE.md` or its sibling code's
    conventions. At most 5 per task; the report says how many were dropped.
  - The report renders them in their own section, *"Odporúčania — framework
    best practices"*, after the findings table and phrased as optional
    suggestions for a future refactor task. The severity legend gains
    ℹ️ info.
  - **One exception stays a real finding.** A feature newer than the installed
    version or the browserslist target (property hooks on PHP 8.2, `defineModel`
    on Vue 3.3, `:has()` outside the target browsers) will not run, so it is
    reported as a regular finding under `ui_ux`, `performance` or `security`
    with a real severity — never under the advisory label, which would hide a
    defect.
  - `framework` joins the `--dimensions=` values and the
    `test_skill.review_dimensions` default. An existing config is migrated once:
    `framework` is appended only to the untouched 1.1.x default list; a
    customised list (a subset, another order, `[]`) is left alone. The migration
    records itself in `test_skill.migrations`, so a user who later removes
    `framework` on purpose does not get it back on the next run.

### Fixed

- **An explicit `false` in the config was switched back on at every run.**
  The Step 2.5 migration set boolean defaults with `(.x //= true)`, and jq's
  `//` treats `false` exactly like a missing key — so `test_skill.time_log`,
  `suggest_missing_acceptance_criteria`, `negative_control` and
  `tick_acceptance_criteria` were rewritten from `false` to `true` on every
  run. Opting out of time logging or of ticking criteria in Teamwork never
  stuck. Defaults now apply only to a missing / `null` value
  (`|= if . == null then true else . end`), the same rule teamwork-task 1.5.0
  uses for the shared config.
- **Comments were never fetched.** Step 3 fetched them only
  `If commentsCount > 0`, and v3 task objects have no `commentsCount` key at
  all, so the gate never opened. Acceptance criteria moved into the last comment
  were never tested. Comments are now always fetched; it is one cheap GET per
  task.
  *Repro:* `jq '.task.commentsCount' <<<"$(curl … /projects/api/v3/tasks/45198800.json)"`
  → `null` on a task with 8 comments.
- **The comments request itself was invalid.** `orderBy=postedAt` is answered
  with **HTTP 400** `orderBy: unknown comment sort.`, the status was not checked,
  and the error body was read as an empty comment list. The request now uses
  `orderBy=date&orderMode=asc` (chronological), checks the HTTP status inline,
  prints a `⚠` naming the endpoint, and falls back to the classic v1 endpoint
  (`GET /tasks/{id}/comments.json`, sorted by `datetime`) with a field map that
  normalises it to the v3 names. When both fail, the report says *"comments
  unavailable"* — never *"no comments"*.
  *Repro:* `curl -w '%{http_code}' …/tasks/45198800/comments.json?orderBy=postedAt` → `400`.
- **Wrong comment field names.** A comment's timestamp is `postedDateTime`;
  there is no `postedAt`. The author is `postedByUserId` (names via
  `include=users`), the body `htmlBody`, the files `files[]`. "Freshest comment"
  cannot be picked by a field that does not exist.
- **`projectId` was read from a key v3 does not return.** The task's project id
  lives at `.tasklist.meta.projectId`; the skill now reads
  `.projectId // .tasklist.meta.projectId`. Without it the opt-in board moves
  (Step 9.4) found no workflow and did nothing. The Step 3 field list now names
  only keys v3 actually returns (`dueDate`, not `dueAt`; `tagIds`).
- **The tolerant-parse contract had the wrong diagnosis, and its own snippets
  re-broke the JSON.** It said Teamwork's WYSIWYG emits raw control characters
  inside JSON strings. The raw body of the same task parses cleanly with
  `printf '%s' "$RAW" | jq`, with `jq … <<<"$RAW"`, and with Python's *strict*
  `json.loads`. The control characters were produced by **zsh's builtin
  `echo`**, which expands the `\n` escapes inside the JSON — and both recipes in
  the block ended in `echo "$CLEAN" | jq`, undoing the Python re-escape on the
  very next line. The block is now the **JSON-through-echo contract**: never
  pass JSON through `echo`; use a here-string or `printf '%s\n'`.
  `json.loads(strict=False)` stays as belt-and-braces, its output also passed on
  through a here-string.
  *Repro (zsh):* `echo "$RAW" | jq .task.name` → `Invalid string: control
  characters from U+0000 through U+001F must be escaped`; `jq .task.name <<<"$RAW"`
  → the task name.
- **The reachability scan aborted in zsh on projects without `wamesk/`.** The
  screen enumeration and the relation lookup in Step 6.6.4 used the glob
  `wamesk/*/src`. An unmatched glob is a hard error in zsh that aborts the whole
  command list, so the screen list came back empty. Both now grep the
  directories recursively and filter the paths afterwards.
  *Repro (zsh, no `wamesk/`):* `{ echo a; grep -rl x wamesk/*/src; echo b; } 2>/dev/null`
  → prints only `a`.
- **The byte-exact tick (Step 9.3) did not say how to load the original
  description.** Through `echo` every escaped newline is rewritten; through
  `jq -r` a trailing newline is added — either way the length assertion compares
  against a corrupted original. It is now extracted with `jq -j … <<<"$BODY"`
  into a file, read with `newline=''`, and the round trip is compared with `cmp`.
- **Step 4 read the acceptance criteria from the wrong block of a canonical
  WAME description.** It split on the first HR, but the format the sibling
  plugins write (`[preamble] → HR → Akceptačné kritériá → HR → Cieľ → …`) puts
  the reporter's preamble above it — or nothing, when the description starts
  with the HR — so the preamble was tested as the criteria (or Step 4.2 drafted
  "missing" ones) while the real criteria, and the `### Prierezové požiadavky`
  block the sibling plugins now add, were never verified. Step 9.3 already
  located the block by its heading. When an `## Akceptačné kritériá` /
  `## Acceptance criteria` heading exists, Step 4 now reads the same block —
  heading to next HR; other descriptions keep the first-HR split.
- **A shell portability contract** now opens `SKILL.md` — no JSON through
  `echo`, inline HTTP checks, no silent `2>/dev/null || echo 0`, no word
  splitting, `=` not `==`, no `${!…}`, no bare globs — so the next edit does not
  reintroduce any of the above.

## [1.1.0] — 2026-09-22

### Added

- **Four cross-cutting review dimensions (new Step 6.6).** A QA pass used to
  answer only the question the task author thought to ask. It now also reviews
  the task's own diff for **UI/UX & accessibility**, **performance**,
  **security** and **reachability**, and reports those findings in their own
  severity-tagged table, separate from the acceptance-criteria table. Pick a
  subset with `--dimensions=ui_ux,security` or turn them off with
  `--dimensions=none`. A dimension that could not run is named in the report:
  a silently skipped dimension reads as a clean bill of health, which is the one
  thing it must never do.
  - *UI/UX* covers the checks that are invisible in a screenshot: a disabled
    control must carry `aria-disabled` **and** a reason, every interactive
    element needs an accessible name, and every new translation key must resolve
    to text rather than to itself — the silent failure where a framework walks a
    dotted key segment by segment, finds a string at the prefix, and renders the
    raw key.
  - *Performance* compares real LCP / CLS / INP / TTFB against configurable
    budgets and reads the diff for the patterns that only hurt at scale (N+1, an
    unchunked batch, a full table walked in PHP) — stating the row count at which
    each one starts to matter, and whether the breach is caused by this change or
    predates it.
  - *Security* covers authorization and tenant isolation on new routes, mass
    assignment, injection through author-controlled strings, secrets, and
    unhandled error paths on reachable routes. It also names the trap that a
    nicely-worded exception message is shown to the user as a generic 500 once
    `APP_DEBUG=false`, so a named exception helps the log, not the screen.
  - *Reachability* answers the question acceptance criteria almost never ask: is
    every registered screen reachable by **clicking**? It enumerates screens from
    the filesystem, collects the live menu links from the rendered navigation,
    subtracts the ones reachable through a parent's relation tab, and reports
    what is left as findable only by typing the URL. `allow_orphans` is the
    escape hatch for deliberate URL-only pages.
- **Negative control for every green test (new Step 6.5.5).** A passing test is
  not evidence until it has been seen to fail. For each criterion verified by a
  test written or changed for the task, the skill applies the smallest possible
  revert of the fix, confirms the test goes red, then restores from a backup kept
  **outside** the repository. A test that passes with and without the fix gets
  its criterion **downgraded to ⚠️ partial** — that case is the single most
  common way a QA report lies. Restoring the file is a hard requirement, verified
  with `git status --porcelain`; a dirty tree left behind is a blocker, not a
  warning. Disable with `--negative-control=false`.
- **Ticking met acceptance criteria in the Teamwork description (new Step 9.3).**
  Criteria that ended ✅ get their `- [ ]` flipped to `- [x]`. Only ✅ is ticked —
  a checklist that claims more than was proven is worse than one nobody filled
  in. On by default; disable with `--tick-ac=false`.
  The rewrite is bounded by a **byte-exact safety contract**, because a Teamwork
  description holds things that no diff of the rendered view shows: an inline
  image is *not* an attachment, it lives only as a link in the markdown, so a
  rewrite that drops it deletes the screenshot with no copy to restore from.
  Regenerating the description from a parsed model is therefore forbidden. The
  only legal edit is the six characters of a checkbox, and the skill must prove
  that before sending — length unchanged, asset markers unchanged, and with all
  checkboxes normalised the two texts identical. The round trip is re-fetched and
  compared, because Teamwork's WYSIWYG has been known to rewrite a submitted
  body, so HTTP 200 is not evidence. Any failed assertion stops the write instead
  of retrying with a different body.

### Changed

- The skill's stated contract is now "read-only for anything destructive". Two
  writes happen by default: the time log (as before) and the acceptance-criteria
  ticks (new, and provably minimal). Board moves, comments and task completion
  remain opt-in.
- `ac.json` gained `review_findings`, `dimensions_run` and `dimensions_skipped`.
  The last two are what make an empty findings array meaningful — without them,
  "no findings" and "nobody looked" are indistinguishable to whatever reads the
  file next. Test evidence gained a `negative_control` field.
- The per-task report gained two sections: the review-findings table and a
  negative-control table.

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
