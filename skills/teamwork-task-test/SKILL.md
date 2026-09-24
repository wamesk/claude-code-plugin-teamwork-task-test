---
name: teamwork-task-test
description: "Use when the user provides a Teamwork.com URL (single task or tasklist) and asks to 'test these tasks', 'otestuj tasky z teamworku', 'preveruj zadanie', 'skontroluj akceptačné kritériá', 'spusti testy pre tasky', 'skontroluj UI/UX, performance a security', 'over či sú všetky podstránky prístupné', or invokes '/teamwork-task-test'. Fetches tasks via the Teamwork REST API v3 (reusing the shared config/token from the `teamwork-task` plugin), parses acceptance criteria from the task description, detects the project's test stack (Pest, PHPUnit, Laravel Dusk, Cypress, Playwright, Selenium, Vitest, Jest), tries to map criteria onto existing tests and run them, optionally drives a real browser via the chrome-devtools MCP for visual verification, and writes a per-task report with each acceptance criterion individually marked as ✅ verified by test / ⚠️ partial / ❌ failed / 📋 manual / 📝 missing-or-proposed. On top of the stated criteria it always runs four cross-cutting review dimensions over the task's own diff — UI/UX and accessibility, performance, security, and page reachability (every registered screen must be reachable from a menu link or a relation tab, never only by typing the URL) — and reports findings the acceptance criteria never asked about. An advisory fifth dimension, framework best practices, only recommends the idiomatic features of the framework versions the project actually has installed (Laravel, PHP, Nova, Vue, Tailwind, CSS/JS per browserslist) as optional refactor suggestions — it never edits code and never fails a criterion. Proves each passing test is load-bearing with a negative control: revert the fix, watch the test fail, restore. Ticks the `- [ ]` boxes of met acceptance criteria to `- [x]` in the Teamwork description under a byte-exact safety contract that touches nothing else, inline images included. Always use this skill when the user wants to verify or QA work captured in a Teamwork task without having to write or run the tests by hand themselves."
argument-hint: "<teamwork-url> [--run-tests=auto|never|always] [--visual=auto|browser|skip] [--dimensions=ui_ux,performance,security,reachability,framework] [--negative-control=true|false] [--tick-ac=true|false] [--comment-on-task=true|false] [--time-log=true|false] [--language=sk|en]"
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

**A QA pass is not only a checklist pass.** Acceptance criteria say what the
author thought to ask for. They are silent about the greyed-out button that
gives no reason, the page nobody can reach, the query that runs once per row,
and the crafted request that answers 500. So on top of the stated criteria this
skill always runs four **cross-cutting review dimensions** over the task's own
diff — **UI/UX & accessibility, performance, security, reachability** (Step 6.6)
— and reports what it finds there as first-class output, separate from the AC
table. A run where every AC is ✅ and the dimensions are empty is a different
claim from a run where nobody looked.

A fifth dimension, **framework best practices** (Step 6.6.5), is **advisory
only**. It reads the same diff against the framework and language versions the
project actually has installed and *recommends* the current idiom where the diff
hand-rolls something the framework ships or uses a pattern that version
replaced. It never edits code, never fails or downgrades a criterion, never
blocks, and never counts toward the result — the tester does not modernize
working code; it leaves a note for a future refactor task.

It also refuses to take a green test at face value: every test that a criterion
leans on is put through a **negative control** (Step 6.5.5) — revert the fix,
confirm the test goes red, restore. A test that passes both before and after
the change proves nothing, and that is the single most common way a QA report
lies.

This skill is **read-only against Teamwork by default** for anything
destructive — it does not move the task on the board, does not complete it, and
does not post a comment unless the user opts in. Two things *are* written back
by default:
- a **time log** (QA is real work and should appear in the timesheet) — off with `--time-log=false`;
- the **`- [x]` ticks** of acceptance criteria that were met (Step 9.3) — off with
  `--tick-ac=false`. This rewrite is bounded by a byte-exact safety contract: it
  may only flip `- [ ]` to `- [x]` inside the acceptance-criteria block and must
  leave the rest of the description, inline images included, identical. It
  refuses to write if it cannot prove that.

---

## Shell portability contract

Every `bash` block in this file runs in the user's login shell — **zsh on
macOS** (Claude Code's Bash tool is `/bin/zsh`), bash elsewhere — and every
Bash tool call is a fresh shell, so no function or variable survives from one
snippet to the next. Keep each snippet valid in both shells:

- **Never pass JSON through `echo`.** zsh's builtin `echo` expands `\n`, `\t`
  and `\\` inside the JSON into raw control characters and `jq` rejects the
  result. Use `jq … <<<"$JSON"` or `printf '%s\n' "$JSON" | jq …`. The same
  goes for description or comment text piped into `grep`, `sed` or `python3`.
  (`echo "$URL" | sed` is harmless — a URL has no backslashes.)
- **Check the HTTP status of every GET** (`curl -w '\n%{http_code}'`) and
  print a `⚠` naming the endpoint on a non-200 or a parse error, then continue
  with an explicit, reported fallback. **Never** `2>/dev/null || echo 0` after
  a parse — that is how a failed request turns into a silent "0 comments".
- **No word splitting in zsh:** `for X in $VAR` iterates once. Iterate lines
  with `while IFS= read -r X; do …; done <<<"$VAR"`.
- `[ "$a" = "$b" ]` with a single `=` (`==` fails in zsh); no `${!ARR[@]}` or
  `${!name}`; no numeric array subscripts (zsh is 1-based) — use `${ARR[@]:$i:1}`.
- **An unmatched glob is a hard error in zsh** and aborts the whole command
  list (e.g. `wamesk/*/src` in a project without `wamesk/`). Quote URLs, grep a
  directory recursively and filter the paths, or use `find`.
- Quote any word that starts with `=` (zsh expands `=word` to a command path).
- Tool calls share no functions: repeat the inline status check in each
  snippet rather than defining a helper in one call and using it in another.

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
- `--dimensions=<csv>` — which cross-cutting review dimensions to run in Step 6.6. Any subset of `ui_ux,performance,security,reachability,framework`, or `none` to run none (default: all five; `framework` is advisory — recommendations only, Step 6.6.5)
- `--negative-control=true|false` — prove each passing test is load-bearing by reverting the fix and watching it fail (default: `true`)
- `--tick-ac=true|false` — tick met acceptance criteria to `- [x]` in the Teamwork description (default: `true`)
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
ENTITY_ID=$(echo "$URL"   | sed -nE 's#.*/(tasks|tasklists)/([0-9]+).*#\2#p')
URL_KIND=$(echo "$URL"    | sed -nE 's#.*/(tasks|tasklists)/[0-9]+.*#\1#p' | sed 's/s$//')
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
7. **Apply CLI flag overrides** (`--run-tests`, `--visual`, `--dimensions`, `--negative-control`, `--tick-ac`, `--comment-on-task`, `--time-log`, `--language`) to the in-memory config — do not persist.
8. **Never echo the API token.** Always pass auth to `curl` via `-u` (kept out of `ps`), never in the URL.

### Step 2.5 — Test-skill config defaults (idempotent merge)

Add this skill's own keys to the same config file, defaulting any missing keys:

```bash
jq '
  (.test_skill //= {}) |
  (.test_skill.run_tests //= "auto") |
  (.test_skill.visual_mode //= "auto") |
  (.test_skill.comment_on_task //= false) |
  (.test_skill.time_log |= if . == null then true else . end) |
  (.test_skill.move_on_pass //= "") |
  (.test_skill.move_on_fail //= "") |
  (.test_skill.report_language //= (.default_language // "sk")) |
  (.test_skill.suggest_missing_acceptance_criteria |= if . == null then true else . end) |
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
  (.test_skill.report_output_dir //= "./.teamwork-task-test") |

  # --- 1.1.0: cross-cutting review dimensions (Step 6.6) -------------------
  # 1.2.0 adds the advisory `framework` dimension to the default. `//=` keeps
  # an existing list as it is, an explicit `[]` ("none") included.
  (.test_skill.review_dimensions //= ["ui_ux", "performance", "security", "reachability", "framework"]) |
  # 1.2.0 one-shot migration: append "framework" ONLY to the untouched 1.1.x
  # default list — a customised list (a subset, another order, `[]`) is a
  # deliberate choice and stays as it is. The marker makes it run once, so a
  # user who later removes "framework" on purpose does not get it back.
  (if ((.test_skill.migrations // []) | index("1.2.0:review_dimensions+framework")) != null then .
   else
     (if .test_skill.review_dimensions == ["ui_ux", "performance", "security", "reachability"]
      then .test_skill.review_dimensions += ["framework"] else . end)
     | .test_skill.migrations = ((.test_skill.migrations // []) + ["1.2.0:review_dimensions+framework"])
   end) |
  (.test_skill.negative_control |= if . == null then true else . end) |
  (.test_skill.tick_acceptance_criteria |= if . == null then true else . end) |
  (.test_skill.performance_budgets //= {
    "lcp_ms": 2500,
    "cls": 0.1,
    "inp_ms": 200,
    "ttfb_ms": 800
  }) |
  (.test_skill.reachability //= {
    "enabled": true,
    "menu_selector_hint": "nav a[href]",
    "screen_globs": [
      "app/Nova/**/*.php",
      "app/Filament/**/Resources/*.php",
      "resources/js/Pages/**/*.{vue,tsx,jsx}",
      "src/pages/**/*.{tsx,jsx,vue}",
      "src/app/**/page.{tsx,jsx}"
    ],
    "relation_field_markers": [
      "HasMany", "HasManyThrough", "BelongsToMany", "MorphMany", "HasOne", "MorphToMany"
    ],
    "allow_orphans": []
  }) |
  (.test_skill.a11y_checks //= [
    "disabled_control_has_reason",
    "disabled_control_is_announced",
    "interactive_element_has_accessible_name",
    "translation_key_resolves_to_text",
    "form_control_has_label",
    "image_has_alt"
  ])
' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
chmod 600 "$CONFIG_FILE"
```

CLI flags override `.test_skill.*` values for the duration of this run.

`test_skill.migrations` records the one-shot migrations that already ran (today
only `1.2.0:review_dimensions+framework`). Leave it alone: deleting an entry
re-runs that migration once, and it still touches only an untouched default.

`reachability.allow_orphans` is the escape hatch for a screen that is
**deliberately** reachable by URL only (a debug page, a legal text linked from
an e-mail, a deep-link landing page). List its identifier there and Step 6.6.4
stops reporting it, so the finding list stays signal and does not train the
reader to ignore it.

---

## Step 3 — Fetch tasks via Teamwork REST API v3

Same auth pattern as `teamwork-task`:

```bash
TOKEN=$(jq -r '.teamwork.api_token' "$CONFIG_FILE")
BASE=$(jq  -r '.teamwork.base_url'  "$CONFIG_FILE")
AUTH="${TOKEN}:xxx"
```

Every GET below checks its HTTP status inline and parses the body with a
here-string — see the **Shell portability contract** at the top. Tool calls do
not share shell functions, so the check is repeated in each snippet. Variables
do not survive between calls either: run the three credential lines above (and
set `ENTITY_ID` / `TASK_ID`) in the **same** Bash call as each snippet.

**Single task** (`URL_KIND=task`):
```bash
# --- Step 3: single task ----------------------------------------------------
URL="${BASE}/projects/api/v3/tasks/${ENTITY_ID}.json"
RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
if [ "$HTTP" != "200" ]; then
  echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null)" >&2
fi
# v3 task objects carry NO top-level `projectId` (and no `commentsCount`) — the
# project id lives on the embedded tasklist reference.
PROJECT_ID=$(jq -r '.task.projectId // .task.tasklist.meta.projectId // empty' <<<"$BODY")
echo "task ${ENTITY_ID}: project ${PROJECT_ID:-⚠ unknown}"
```

**Tasklist** (`URL_KIND=tasklist`):
```bash
# --- Step 3: tasklist -------------------------------------------------------
URL="${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json"
RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
if [ "$HTTP" != "200" ]; then
  echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null)" >&2
fi
PROJECT_ID=$(jq -r '.tasklist.projectId // empty' <<<"$BODY")

# One task per line (compact JSON) so later snippets can read it without echo.
TASKS_FILE="/tmp/tw_test_tasks_${ENTITY_ID}.jsonl"; : > "$TASKS_FILE"
PAGE=1
while :; do
  URL="${BASE}/projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?pageSize=100&page=${PAGE}"
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" != "200" ] || ! jq -c '.tasks[]' <<<"$BODY" >> "$TASKS_FILE"; then
    echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null) — the task list is INCOMPLETE from page ${PAGE} on" >&2
    break
  fi
  [ "$(jq -r '.meta.page.hasMore // false' <<<"$BODY")" = "true" ] || break
  PAGE=$((PAGE + 1))
done
echo "tasklist ${ENTITY_ID}: $(wc -l < "$TASKS_FILE" | tr -d ' ') task(s), project ${PROJECT_ID:-⚠ unknown}"
```

Extract per task (`.task` for a single task, one line of `$TASKS_FILE` per task
in a tasklist):
- `{id, name, description, status, priority, estimateMinutes, dueDate, tasklistId, tagIds, assigneeUserIds, workflowStages}` — these are the keys v3 actually returns.
- **`projectId` = `.projectId // .tasklist.meta.projectId`.** v3 has no
  top-level `projectId` (it reads `null`); the value sits on the embedded
  tasklist reference. The opt-in board moves (Step 9.4) need it to find the
  project's workflow.
- There is **no `commentsCount`** either — nothing may be gated on it.

By default skip tasks with `status == "completed"` unless the user passed `--include-completed` (allow this flag implicitly — it is a sensible escape hatch for QA passes on already-finished work).

**Always fetch the comments** (v1.2.0: no count gate — v3 never returns
`commentsCount`, so a `commentsCount > 0` gate never opened, and one GET per task
is cheap next to a QA pass that misses the criteria somebody moved into the last
comment). Teamwork often holds the freshest acceptance criteria in the last
comment, so the order matters: `orderBy=date&orderMode=asc` is chronological
(`orderBy=postedAt` is rejected with HTTP 400 *"orderBy: unknown comment
sort."*).

```bash
# --- Step 3: comments (v3, v1 fallback) -------------------------------------
# TASK_ID = the task being processed ($ENTITY_ID for a single-task URL).
# One comment per line, oldest first, normalised to the v3 field names:
# {id, postedDateTime, postedByUserId, author, htmlBody, files}.
TASK_ID=${TASK_ID:-$ENTITY_ID}
COMMENTS_FILE="/tmp/tw_test_comments_${TASK_ID}.jsonl"; : > "$COMMENTS_FILE"
COMMENTS_SRC=v3; PAGE=1
while :; do
  URL="${BASE}/projects/api/v3/tasks/${TASK_ID}/comments.json?pageSize=100&page=${PAGE}&orderBy=date&orderMode=asc&include=users"
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" != "200" ] || ! jq -c '(.included.users // {}) as $u | .comments[]
        | {id, postedDateTime, postedByUserId,
           author: (($u[(.postedByUserId|tostring)] // {}) | [.firstName, .lastName] | map(select(. != null and . != "")) | join(" ")),
           htmlBody, files: (.files // [])}' <<<"$BODY" >> "$COMMENTS_FILE"; then
    echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null) — falling back to the v1 comments endpoint" >&2
    COMMENTS_SRC=v1; break
  fi
  [ "$(jq -r '.meta.page.hasMore // false' <<<"$BODY")" = "true" ] || break
  PAGE=$((PAGE + 1))
done

if [ "$COMMENTS_SRC" = "v1" ]; then
  # v1 → v3 field map: datetime → postedDateTime · html-body (plain: body) →
  # htmlBody · author-id → postedByUserId · author-firstname/-lastname → author ·
  # attachments → files. v1 server order is unspecified — sort by datetime.
  # Page until an EMPTY page: v1 may cap pageSize below the 100 requested, so a
  # short page does not prove it was the last one.
  : > "$COMMENTS_FILE"; PAGE=1
  while :; do
    URL="${BASE}/tasks/${TASK_ID}/comments.json?page=${PAGE}&pageSize=100"
    RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
    HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
    N=$(jq -r '.comments | length' <<<"$BODY" 2>/dev/null)
    if [ "$HTTP" != "200" ] || [ -z "$N" ] || ! jq -c '.comments[]
        | {id: (.id|tonumber? // .), postedDateTime: .datetime, postedByUserId: (."author-id"|tonumber? // .),
           author: ([."author-firstname", ."author-lastname"] | map(select(. != null and . != "")) | join(" ")),
           htmlBody: (."html-body" // .body), files: (.attachments // [])}' <<<"$BODY" >> "$COMMENTS_FILE"; then
      echo "⚠ GET $URL → HTTP $HTTP — comments UNAVAILABLE for task ${TASK_ID}; report them as unavailable, never as \"no comments\"" >&2
      COMMENTS_SRC=unavailable; : > "$COMMENTS_FILE"; break
    fi
    [ "$N" -eq 0 ] && break
    PAGE=$((PAGE + 1))
  done
  [ "$COMMENTS_SRC" = "v1" ] && jq -sc 'sort_by(.postedDateTime)[]' "$COMMENTS_FILE" > "$COMMENTS_FILE.tmp" && mv "$COMMENTS_FILE.tmp" "$COMMENTS_FILE"
fi
if [ "$COMMENTS_SRC" = "unavailable" ]; then
  echo "task ${TASK_ID}: comments UNAVAILABLE"
else
  echo "task ${TASK_ID}: $(wc -l < "$COMMENTS_FILE" | tr -d ' ') comment(s) via ${COMMENTS_SRC}"
fi
```

| Datum | v3 comment | v1 comment |
| --- | --- | --- |
| time | `postedDateTime` (there is **no** `postedAt`) | `datetime` |
| HTML body | `htmlBody` | `html-body` (plain: `body`) |
| author | `postedByUserId` (id only; names via `include=users` → `.included.users`) | `author-id`, `author-firstname`, `author-lastname` |
| files | `files[]` | `attachments[]` |

`COMMENTS_SRC=unavailable` is a different result from an empty file: the report
says *"comments unavailable (HTTP …)"* for that task, never *"no comments"* —
the two read the same otherwise, and only one of them is true.

If a fetch returns HTTP 401 → token invalid; re-prompt the user (re-run Step 2's first-run flow). 403/404 on the task or tasklist → report and stop.

> **JSON-THROUGH-ECHO CONTRACT (critical).** Up to 1.1.0 this block was called
> the *tolerant-parse contract* and blamed Teamwork: it said the WYSIWYG emits
> raw control characters inside JSON string values, so the curl body had to be
> sanitised before `jq`. **That diagnosis was wrong.** Re-checked on 2026-09-24
> against task 45198800, whose description and comments are full of line
> breaks: the raw body parses cleanly with `printf '%s' "$RAW" | jq`, with
> `jq … <<<"$RAW"`, and even with Python's *strict* `json.loads` — Teamwork
> escapes its control characters correctly. The raw control characters were
> almost certainly produced **by the shell**: Claude Code runs these snippets in
> zsh, and zsh's builtin `echo` expands every `\n` / `\t` / `\\` escape inside
> the JSON into a real control byte, so `echo "$RAW" | jq` dies with
> `Invalid string: control characters from U+0000 through U+001F must be escaped`.
> The old sanitize recipes could not help, because each ended in
> `echo "$CLEAN" | jq`: Python re-escaped the JSON correctly and the very next
> `echo` broke it again.
>
> The contract is therefore: **never pass JSON through `echo`.**
>
> ```bash
> RAW=$(curl -sS -u "$AUTH" -H "Accept: application/json" "$URL")
> jq -r '.task.name' <<<"$RAW"                 # here-string — byte-exact
> printf '%s\n' "$RAW" | jq -r '.task.name'    # equivalent
> # NEVER: echo "$RAW" | jq …                  # zsh turns \n inside strings into raw newlines
> ```
>
> Keep the tolerant re-emit as **belt-and-braces** for a body that really does
> carry raw control characters (e.g. text that reached you from some other
> source). It is harmless on a clean body, and its output goes on through a
> here-string too:
>
> ```bash
> # json.loads(strict=False) accepts raw control characters inside strings;
> # json.dump re-escapes them losslessly.
> CLEAN=$(printf '%s' "$RAW" | python3 -c 'import sys,json; json.dump(json.loads(sys.stdin.read(), strict=False), sys.stdout)')
> jq -r '.task.name' <<<"$CLEAN"
> ```
>
> Without `python3`, the lossy last resort is `CLEAN=$(printf '%s' "$RAW" | tr
> '\000-\037' ' ')` followed by `jq … <<<"$CLEAN"` — it collapses every control
> byte, newlines included, to a space. Description and comment text piped into
> `grep`, `sed` or `python3` follows the same rule: `printf '%s'` or a
> here-string, never `echo`. A parse failure must never pass silently as an
> empty result — print a `⚠` naming the endpoint, as the snippets above do.

---

## Step 4 — Parse acceptance criteria from the task description

**Canonical WAME format first.** When the description has an
`## Akceptačné kritériá` / `## Acceptance criteria` heading — the format
`teamwork-task-analyze`, `-from-desk`, `-from-session` and `-from-dnr` write
(`[preamble] → HR → AC → HR → Cieľ → HR → Technický popis`) —
`acceptance_criteria` is the block from that heading to the next HR, including
its `### Prierezové požiadavky` / `### Cross-cutting requirements` sub-block
(each item there is an ordinary criterion; a key in parentheses such as
`(security)` names the Step 6.6 dimension it overlaps — `security` /
`performance` give the same `kind`, `ui_ux` / `reachability` give `ui`; the
advisory `framework` dimension never appears there — the sibling plugins keep it
in the technical plan, because a recommendation cannot be ticked), and
`final_summary` is everything below that HR. Text above the first HR is the
reporter's preamble — context, not criteria. This is the same block Step 9.3
ticks; the plain first-HR split below would take the preamble (or nothing, when
the description starts with the HR) as the criteria. Otherwise:

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

Pick the highest-scoring candidate. Decide the runner.

> **SHELL-SAFETY CONTRACT (critical).** The `<test name>` is grepped verbatim
> from the test file and is fully attacker/author-controlled. **Never** splice
> it into a double-quoted argument (`--filter "<test name>"`): a name containing
> a double quote followed by `$(…)` or backticks closes the quote and executes
> arbitrary shell. Always pass it as a **single-quoted, escaped** literal.
> Build the escaped form once with `printf %q` (or wrap in single quotes and
> escape embedded single quotes) and interpolate that, e.g.:
>
> ```bash
> TEST_NAME='it sends "$(rm -rf ~)" mail'      # whatever was grepped, verbatim
> Q=$(printf '%q' "$TEST_NAME")                # shell-safe, no eval surface
> ./vendor/bin/pest --filter "$Q"              # $Q already fully escaped
> ```
>
> In the runner table below, every `<test name>` / `<file>` placeholder MUST be
> substituted with such a `printf %q`-escaped value, not the raw string.

| Test file path                                  | Runner command (default — override via `test_runners.*` config)        |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| `tests/Feature/…` / `tests/Unit/…` (Pest)       | `./vendor/bin/pest --filter "$Q"`                                      |
| `tests/Feature/…` / `tests/Unit/…` (PHPUnit)    | `./vendor/bin/phpunit --filter "$Q"`                                   |
| `tests/Browser/…` (Laravel Dusk)                | `php artisan dusk --filter "$Q"`                                       |
| `cypress/e2e/…`                                 | `npx cypress run --spec "$QFILE" --headless`                           |
| `tests/e2e/…` / `playwright/…` (Playwright)     | `npx playwright test "$QFILE" -g "$Q"`                                 |
| `tests/**/*.{test,spec}.{ts,js,tsx,jsx}` (Vitest) | `npx vitest run -t "$Q" "$QFILE"`                                      |
| Same path family but project uses Jest          | `npx jest --testNamePattern "$Q" "$QFILE"`                             |

where `Q=$(printf '%q' "$TEST_NAME")` and `QFILE=$(printf '%q' "$TEST_FILE")` are
pre-escaped once before the runner is invoked.

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

### 6.5.5 Negative control — prove the test is load-bearing

**A green test is not evidence until you have seen it go red.** The most common
way a QA report lies is by pointing at a test that passes with the fix *and
without* it. That happens constantly and innocently: the test asserts a
behaviour the code already had, the assertion is vacuous (`// Assert` with
nothing under it), the fixture never reaches the changed branch, or the runner
silently skipped the case.

So for every criterion that ended `✅ verified` **by a test written or changed
for this task**, run a negative control. Skip it when
`config.test_skill.negative_control == false`, when the evidence is a
pre-existing untouched test (there is no "before" to revert to), or when the
verdict came from the browser rather than a test.

```bash
# 1. Identify the smallest edit that undoes the fix for this criterion. Prefer a
#    ONE-TOKEN change over deleting a block — it is easier to restore exactly:
#      * narrow a catch:      catch (Throwable $e)   -> catch (DomainException $e)
#      * drop a null guard:   {{ $x['k'] ?? '' }}    -> {{ $x['k'] }}
#      * invert a condition:  @if ($canDoThing)      -> @if (true)
#      * rename an attribute: aria-disabled="true"   -> data-x="1"
#
# 2. Back the file up OUTSIDE the repo first. Never rely on `git checkout` to
#    restore it: another session may hold uncommitted work in the same checkout,
#    and checkout would take that with it.
cp "$FILE" "/tmp/nc_$(basename "$FILE")"

# 3. Apply the minimal revert, run ONLY the tests for this criterion.
perl -i -pe 's/<fixed>/<broken>/' "$FILE"
./vendor/bin/pest --filter "$Q"        # expect FAILURE

# 4. Restore from the backup and confirm the tree is clean again.
cp "/tmp/nc_$(basename "$FILE")" "$FILE"
rm -f "/tmp/nc_$(basename "$FILE")"
git status --porcelain -- "$FILE"      # expect EMPTY
```

Verdict mapping:

| Negative control result | What it means | Criterion status |
| --- | --- | --- |
| Test fails when reverted | The test really covers the fix | stays `✅ verified`, evidence gains `negative_control: confirmed` |
| Test still passes when reverted | The test does not exercise the change | **downgrade to `⚠️ partial`** and say so: *"test passes with and without the fix — it does not cover this criterion"* |
| Could not construct a minimal revert | Honest unknown | stays `✅ verified`, evidence gains `negative_control: not_attempted` with the reason |

**Step 4 of the recipe is not optional.** Leaving a reverted file behind turns a
QA pass into a broken working tree, and the user will discover it as a mystery
failure hours later. Verify the restore with `git status --porcelain` on that
exact path, and if the check is not empty, say so loudly in the report rather
than moving on.

---

## Step 6.6 — Cross-cutting review dimensions

Runs **once per task**, after all criteria have been resolved. This is the part
of the QA pass that looks at what the acceptance criteria did **not** ask about.
Findings here never change an AC's status — they are reported in their own
section, each with a severity (`high` / `medium` / `low`) and a concrete
location. The advisory `framework` dimension (6.6.5) is different in kind: it
produces `info` recommendations only, rendered in a section of their own.

Scope the review to **the task's own diff** so the output stays about this
change rather than becoming a whole-repo audit:

```bash
# Commits carrying this task id, plus anything still uncommitted.
RANGE=$(git log --all --format=%H --grep="\[${TASK_ID}\]" | tail -1)
if [ -n "$RANGE" ]; then
  CHANGED=$(git diff --name-only "${RANGE}~1" HEAD; git diff --name-only HEAD)
else
  CHANGED=$(git diff --name-only HEAD)
fi
CHANGED=$(printf '%s\n' "$CHANGED" | sort -u | grep -v '^$')
```

Run only the dimensions listed in `config.test_skill.review_dimensions` (or the
`--dimensions=` override). Each subsection below says what to look for and what
counts as a finding — do not report a "finding" you cannot point at a line for.

### 6.6.1 UI/UX & accessibility

Applies when `CHANGED` contains anything matching
`config.test_skill.visual_file_patterns` (templates, components, views) or a
class that feeds one.

Check, in this order:

1. **A disabled control must say why.** A greyed-out button or a bare `<span>`
   standing in for one tells a sighted user "this does not work" and tells a
   screen reader *nothing* — an inert `<span>` is announced as ordinary text.
   Require `aria-disabled="true"` (or a real `<button disabled>`) **and** a
   `title` / `aria-describedby` naming the reason. A control the user cannot use
   and cannot understand is a support ticket.
2. **Every interactive element has an accessible name.** Icon-only buttons and
   links need `aria-label` or visible text.
3. **A translation key must resolve to text, not to itself.** Frameworks that
   walk dotted keys segment by segment (Laravel's `Arr::get`, i18next, many
   others) cannot reach `a.b.c` when a **string** already lives at `a.b`. The
   miss is silent: the screen renders the raw key. So for every new translation
   key the diff introduces, assert `__($key) !== $key`, and flag any new key
   whose dotted prefix collides with an existing string key. Put hints in their
   own group (`hint.thing`) rather than hanging them off the thing (`thing.hint`).
4. **Form controls have labels**; **images have `alt`**.
5. **Layout stability and contrast** — if a browser session is available, read
   CLS from the trace (see 6.6.2) and report any control whose disabled state is
   conveyed by colour alone.

### 6.6.2 Performance

Two halves, both cheap:

**Frontend** — when a browser session is available, run
`performance_start_trace` with `reload: true` on the screen the task touches and
compare against `config.test_skill.performance_budgets`:

| Metric | Default budget | Read as |
| --- | --- | --- |
| LCP | 2500 ms | slow first paint of the main content |
| CLS | 0.1 | content jumping under the user's cursor |
| INP | 200 ms | sluggish response to input |
| TTFB | 800 ms | slow server or slow round trip |

State plainly whether a breach is **caused by this change** or is a
pre-existing property of the app shell. A QA report that blames the framework's
CLS on a three-line null-guard commit is noise.

**Backend** — read the diff for the patterns that only hurt at scale:

- a query inside a loop, or a relation touched per row without eager loading (N+1);
- a full-table walk in PHP where a `WHERE` would do — call it out and say at what row count it stops being free;
- a data migration or batch job with no `chunk`/`chunkById`, hydrating a whole table;
- an added index that duplicates an existing one, or a missing index on a new foreign key;
- work done per item that could be done once outside the loop.

For each, state the row count at which it matters. "Fine at 1 430 rows, a full
scan at 100 000, and it is a one-time migration" is a useful finding.
"Potentially slow" is not.

### 6.6.3 Security

Read the diff for the classes that actually ship:

- **Authorization** — a new route, action, or button: is it behind the same
  gate as its neighbours? Does an object-scoped check (`view $thisRecord`) exist,
  or only a role check that any tenant passes (IDOR)?
- **Tenant / company isolation** — does the change reach data through a raw
  query builder, a `DB::` call, or a join that bypasses the global scope?
- **Unhandled error paths on reachable routes.** A hidden or disabled button in
  the UI is not a boundary: the route is still there. An endpoint that answers a
  crafted request with an unhandled exception (HTTP 500) is both an error-tracker
  flood and a weak signal. Catch the named condition and answer deliberately.
- **Mass assignment** — new fillable fields, `$request->all()` into `fill()`.
- **Injection** — interpolation into raw SQL, shell, or a filter/`-g` argument.
  Test names and file paths grepped out of a repo are author-controlled; pass
  them as `printf %q`-escaped literals (same contract as Step 6.2).
- **What the message says** — an exception or validation message returned to the
  user must not carry internals (paths, SQL, IDs of other tenants' rows). Note
  the flip side too: with `APP_DEBUG=false` a nicely-worded exception is shown to
  the user as a generic 500, so a named exception helps the *log*, not the
  screen. If the user is supposed to read it, it has to travel as a flash
  message or a validation error, not as a thrown message.
- **Secrets** — a token, key, or password added to code, a log line, a commit
  message, or a report artefact.

Report severity from reachability and impact, not from how exotic the class is.

### 6.6.4 Reachability — can a user actually get there?

The question the acceptance criteria almost never ask: **is every screen this
app registers reachable by clicking?** A page that exists, is authorized, and
renders fine, but that no menu item and no relation tab points at, is invisible
in practice. It is found by typing the URL — which means it is found by nobody.

```bash
# 1. Every screen the app registers. Framework-specific; the globs live in
#    config.test_skill.reachability.screen_globs. For Laravel Nova, resource
#    classes are the screens.
#    Grep the directories recursively and filter the paths afterwards: a glob
#    such as `wamesk/*/src` is a hard error in zsh when the project has no
#    `wamesk/`, and it aborts the whole command list, scan included. The
#    `2>/dev/null` only hides "No such file or directory" for a directory the
#    project does not have; it is not a parse-error suppression.
grep -rlE "extends (Resource|BaseResource)\b" app/Nova wamesk --include="*.php" 2>/dev/null \
  | grep -E '^(app/Nova/|wamesk/[^/]+/src/)' \
  | sed -E 's|.*/||; s|\.php$||' | grep -v '^BaseResource$' | sort -u > /tmp/screens.txt

#    NOTE: enumerate from the FILESYSTEM, not from the framework registry at
#    runtime. A CLI/tinker process has usually not booted the HTTP service
#    provider that registers them, so the registry comes back almost empty and
#    the whole check silently passes.

# 2. Every screen a menu links to. Read it from the rendered navigation — the
#    live sidebar is the truth, a config array is a guess. With a browser
#    session: navigate to the app root, expand the nav, take_snapshot, and
#    collect the hrefs.

# 3. Anything in (1) and not in (2) is a CANDIDATE orphan, not yet a finding.
comm -23 /tmp/screens.txt /tmp/menu.txt > /tmp/candidates.txt

# 4. A screen legitimately reachable through a parent's relation tab is NOT an
#    orphan. For each candidate, look for a reference from another screen:
while IFS= read -r C; do
  HITS=$(grep -rl "${C}::class" app/Nova wamesk --include="*.php" 2>/dev/null \
         | grep -E '^(app/Nova/|wamesk/[^/]+/src/Nova/)' \
         | grep -v "/${C}.php" | head -3 | sed -E 's|.*/||' | paste -sd ',' -)
  printf "%-40s referenced by: %s\n" "$C" "${HITS:-NOBODY}"
done < /tmp/candidates.txt
```

A candidate is a **finding** only when all of these hold:
- no menu link,
- no relation field on any other screen points at it,
- it is not listed in `config.test_skill.reachability.allow_orphans`,
- and it actually loads (confirm with one navigation — a screen that 403s or
  404s for everyone is a different, larger problem, so say which one it is).

Report it as: *"`<Screen>` is registered and loads (`<url>`, N rows) but nothing
links to it — reachable only by typing the URL."* Then say which it probably is:
a screen that should get a menu entry, a leftover that should be unregistered,
or a deliberate URL-only page that belongs in `allow_orphans`.

Also check the reverse direction, which is the cheaper half: **every menu link
resolves.** A link to a resource that was renamed or unpublished is a 404 the
user meets by clicking, and it is one HTTP HEAD per link to find.

### 6.6.5 Framework best practices — advisory, recommendations only

The question: does the task's diff use the idioms and built-in features of the
framework and language versions **this project actually has installed** — or
does it hand-roll something the framework already ships, or use a pattern that
version replaced? The answer is a short list of **recommendations**, never
defects.

> **ADVISORY CONTRACT (critical).** A tester does not modernize working code.
> This dimension only recommends. It **never** edits code (the only code edit
> this skill makes is the negative control's temporary revert in Step 6.5.5,
> and that is always restored), never applies a recommendation during the run,
> never changes a criterion's status (no downgrade, no ❌), never blocks, and
> never counts toward the task's `Stav testovania`, the Step 8 totals or any
> pass/fail roll-up. Every item is recorded with `"dimension": "framework"`, `"severity": "info"`,
> `"fixed": false` and `"advisory": true`, and is rendered in a report section of
> its own as an optional suggestion for a future refactor task.

**1. Detect the installed versions — never assume them from memory.** Read the
resolved versions from the lock files; a manifest constraint such as `^12.0` is
not a version. Run this once per run and reuse the result for every task. With
Laravel Boost installed (`laravel/boost`), its `application-info` tool gives the
PHP / Laravel / package versions directly.

```bash
# --- Step 6.6.5: installed framework versions --------------------------------
# PHP side: resolved versions from composer.lock (else vendor/composer/installed.json),
# the PHP floor from composer.json. A package repo without either prints its
# composer.json constraints and says that they are constraints.
PHP_PKGS='^(laravel/(framework|nova|boost)|livewire/livewire|inertiajs/inertia-laravel|pestphp/pest|phpunit/phpunit)$'
if [ -f composer.lock ]; then
  jq -r --arg re "$PHP_PKGS" '.packages[]?, ."packages-dev"[]? | select(.name | test($re))
                              | "\(.name) \(.version)"' composer.lock
elif [ -f vendor/composer/installed.json ]; then
  jq -r --arg re "$PHP_PKGS" '(.packages? // .)[] | select(.name | test($re))
                              | "\(.name) \(.version)"' vendor/composer/installed.json
elif [ -f composer.json ]; then
  jq -r --arg re "$PHP_PKGS" '(.require // {}) + (."require-dev" // {}) | to_entries[]
                              | select(.key | test($re))
                              | "\(.key) \(.value) (constraint only, not installed)"' composer.json
fi
if [ -f composer.json ]; then
  jq -r '"php " + ((.config.platform.php // .require.php // "?") | tostring)' composer.json
fi

# JS side: the version installed in node_modules (works for npm, pnpm and yarn),
# else the one resolved in package-lock.json; without either print the
# package.json constraint and say that it is one.
if [ -f package.json ]; then
  while IFS= read -r P; do
    [ -z "$P" ] && continue
    V=""
    if [ -f "node_modules/$P/package.json" ]; then
      V=$(jq -r '.version // empty' "node_modules/$P/package.json")
    elif [ -f package-lock.json ]; then
      V=$(jq -r --arg p "$P" '.packages["node_modules/" + $p].version // .dependencies[$p].version // empty' package-lock.json)
    fi
    if [ -n "$V" ]; then
      printf '%s %s\n' "$P" "$V"
    else
      printf '%s %s (constraint only, not installed)\n' "$P" \
        "$(jq -r --arg p "$P" '.dependencies[$p] // .devDependencies[$p] // "?"' package.json)"
    fi
  done < <(jq -r '(.dependencies // {}) + (.devDependencies // {}) | keys[]
                  | select(test("^(vue|react|nuxt|vite|typescript|tailwindcss|@tailwindcss/.+|@inertiajs/.+|@ionic/.+)$"))' package.json)
  # The target browsers decide which CSS / JS features are safe to use.
  if [ -f .browserslistrc ]; then
    printf 'browserslist %s\n' "$(grep -vE '^[[:space:]]*(#|$)' .browserslistrc | paste -sd ',' -)"
  else
    jq -r '"browserslist " + ((.browserslist // "defaults (none configured)")
           | if type == "object" then (.production // (to_entries[0].value) // []) else . end
           | if type == "array" then join(",") else tostring end)' package.json
  fi
  jq -r '"node (engines) " + (.engines.node // "?")' package.json
fi
if [ -f .nvmrc ]; then printf 'node (.nvmrc) %s\n' "$(cat .nvmrc)"; fi
```

A package constraint printed as *"constraint only, not installed"* is a
**floor**: a recommendation may rely on it, a *newer than installed* finding
(point 4) may not — that needs the resolved version. PHP is the other way
round: `config.platform.php` / `require.php` is the lowest version the code has
to run on, so syntax newer than that floor **is** a point-4 finding, whatever
PHP the developer runs locally. The browserslist target plays the same role for
CSS and JS features. When the snippet prints no version and no constraint for a
stack and Boost is not installed, there is nothing to gate on, so that stack
gets no recommendations; say so in the report.

**2. Look the installed version up in current docs, not in memory.** Model
knowledge of a framework is stale by design. In this order: Laravel Boost
`search-docs` when the project has `laravel/boost` (it searches the docs of the
installed versions); otherwise the context7 MCP (`resolve-library-id` →
`query-docs`); otherwise the official docs via WebFetch
(`laravel.com/docs/<major>.x`, `nova.laravel.com/docs/v<major>`, `vuejs.org`,
`tailwindcss.com/docs`, MDN / caniuse against the browserslist target). Read the
upgrade guide or release notes of the installed major for what is new and what
is deprecated. With no docs source reachable, issue **no** recommendations and
say so in the report — a recommendation from memory is exactly what this
dimension exists to avoid.

**3. Read the diff — only lines the task adds or changes** (the `+` side of
`git diff` over `CHANGED`). A line is worth a recommendation when it:

- hand-rolls something the installed version ships — a `match` over status
  strings where an enum cast exists, manual number / date formatting where
  `Number::` or `Intl.*` does it, a new dependency for a capability the framework
  already has;
- uses a pattern the installed version replaced — a new `$casts` property in
  a Laravel 11+ project whose models already use the `casts()` method (the
  property still works; this is an idiom, not a deprecation), a mixin in a
  Vue 3 `<script setup>` codebase, `props` + `emit('update:modelValue')` where
  `defineModel` exists, v3 `tailwind.config.js` theme extension in a Tailwind v4 project;
- calls an API that is deprecated in the installed version and already emits a
  deprecation warning.

Idioms by stack — illustrations, not a checklist, and always gated by step 1:

| Stack | Worth recommending when the installed version has it |
| --- | --- |
| PHP 8.2–8.4 | readonly classes / properties, enums with methods and interfaces, first-class callable syntax, `#[\Override]` (8.3), typed class constants (8.3), property hooks and asymmetric visibility (8.4), `new` in initializers |
| Laravel 11 / 12 | `casts()` method and enum casts, `Attribute` accessors / mutators, `Model::shouldBeStrict()` / `preventLazyLoading()` outside production, `once()`, `Context`, `defer()` / `Concurrency`, `ShouldBeUnique` and job middleware, `Rule::enum`, `Password::defaults`, `Http::` pool / retry, `lazyById` / `chunkById`, `whereAll` / `whereAny`, `Number` / `Str` helpers, scoped route model binding, `#[Scope]` / `#[ObservedBy]` / `#[UsePolicy]` where available, Pest 3 architecture tests |
| Nova 4 / 5 | `Nova::mainMenu()`, field `dependsOn()`, `->filterable()`, conditional field visibility, `Badge` / `Status` fields, `->copyable()`, repeater fields, `->withoutConfirmation()` only deliberately |
| JS / TS (per browserslist) | optional chaining / `??`, `structuredClone`, `Array.prototype.at` / `toSorted` / `findLast`, `Object.groupBy`, `AbortController` for cancellable fetches, `Intl.*` instead of manual formatting, ES modules |
| Vue 3.4 / 3.5 | `<script setup>` + Composition API, `defineModel`, typed `defineProps` with reactive props destructure (3.5), `useTemplateRef` / `useId` (3.5), composables instead of mixins, `v-bind` in `<style>`, async components / `Suspense` for heavy parts |
| CSS (per browserslist) | custom properties, native nesting, `:has()`, `:is()` / `:where()`, container queries, logical properties, `dvh` / `svh`, `clamp()`, `@layer`, `aspect-ratio`, flex `gap`, `prefers-reduced-motion` / `prefers-color-scheme` |
| Tailwind 3.4 / 4 | v4 CSS-first config (`@import "tailwindcss"`, `@theme`, `@utility`, `@variant`), `size-*`, `@container` variants (core in v4; in v3 only with the `@tailwindcss/container-queries` plugin already installed), `has-*` / `group-has-*`, the `*:` child variant, `motion-safe` / `motion-reduce`, arbitrary values only as a last resort |

**Not a recommendation** — drop it rather than dilute the list:

- *"could be more modern"* without a concrete replacement that the **installed**
  version offers;
- anything outside the task's diff — this is not an audit of untouched code;
- a newer idiom that contradicts the project's `CLAUDE.md` or the established
  convention of the sibling code. Consistency beats novelty: two patterns side
  by side are worse than one older one, unless the task itself is a refactor.

**4. The exception — newer than installed is a bug, not a recommendation.** A
feature that the installed version or the browserslist target does not support
will not run: property hooks on a PHP 8.2 platform, `defineModel` on Vue 3.3,
`Object.groupBy` or `:has()` outside the target browsers, a Tailwind v4
directive in a v3 project, an API the installed version removed. Report it as
a **regular finding** under the dimension it breaks — `ui_ux` (renders wrong or
not at all), `performance` or `security`, whichever fits — with a real
severity and `advisory` false. Never file it under `framework`: the advisory
label would hide a defect. If a criterion's test or browser check already
failed because of it, name the cause in that criterion's detail; the verdict
itself still comes from the Step 6.2 / 6.3 evidence.

**5. What each recommendation says:** the location (`file:line` in the diff),
the installed version it relies on (*Laravel 12.28.1*, *Vue 3.5.13*), what the
code does now, the concrete replacement that version offers, the docs page
checked (URL, or the Boost `search-docs` query), and one line of payoff (less
code, a bug class removed, a deprecation avoided). Phrase it as an optional
suggestion for a future refactor task — *"Zvážiť pri najbližšom refaktoringu: …"*
— never as *"musí"* or *"chyba"*.

**Cap: 5 per task.** Keep the five with the highest payoff — a deprecation
first, then a hand-rolled duplicate of a built-in, then an idiom upgrade — and
report how many were dropped (*"+2 ďalšie odporúčania vynechané (limit 5 na
task)"*). A long list of style nudges trains the reader to skip the section.

### 6.6.6 Recording the findings

Append to the task's `ac.json` under a sibling key, so the AC results stay
clean and machine-readable:

```json
{
  "task_id": 123456,
  "criteria": [ … ],
  "review_findings": [
    {
      "dimension": "ui_ux",
      "severity": "medium",
      "title": "Disabled button gives no reason",
      "location": "resources/views/nova/invoice/buttons-card.blade.php:41",
      "detail": "A greyed <span> with no aria-disabled and no title. A screen reader announces it as plain text; a sighted user cannot tell whether it is a permission problem or an unsupported document type.",
      "in_this_diff": true,
      "fixed": true
    },
    {
      "dimension": "framework",
      "severity": "info",
      "advisory": true,
      "title": "Status mapped by hand instead of an enum cast",
      "location": "app/Models/Invoice.php:34",
      "detail": "A match() over status strings in an accessor. Laravel 12 casts the column to an enum via the casts() method, which removes the string comparison and the invalid-status branch.",
      "installed_version": "laravel/framework v12.28.1",
      "suggestion": "casts(): ['status' => InvoiceStatus::class]",
      "docs": "https://laravel.com/docs/12.x/eloquent-mutators#enum-casting",
      "in_this_diff": true,
      "fixed": false
    }
  ],
  "framework_versions": { "laravel/framework": "v12.28.1", "php": "^8.3", "vue": "3.5.13" },
  "framework_recommendations_dropped": 0
}
```

`in_this_diff` separates *"this change introduced it"* from *"this change
walked past it"* — both are worth reporting, and conflating them is how QA
reports become unreadable. Set `fixed` only if the fix actually landed during
this run, and name the commit in the report when it did.

A `framework` item always carries `"severity": "info"`, `"advisory": true` and
`"fixed": false` — this skill never applies one — and every reader of
`ac.json` (the Step 7 status, the Step 8 totals, a CI gate) must skip
`advisory: true` items when it rolls up pass / fail. `advisory` is absent or
`false` on every other dimension. `framework_versions` records what step 1 of
6.6.5 detected, so a recommendation can be re-checked against the same
versions later; `framework_recommendations_dropped` is the count cut by the cap.

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

#### Nálezy mimo akceptačných kritérií (Step 6.6)

Always render this section, even when it is empty — *"4 dimensions reviewed, no
findings"* is a claim, and its absence is not. The advisory `framework`
recommendations do **not** go into this table; they get their own section below.

| Rozmer | Sev | Nález | Kde | V tomto diffe? |
|--------|-----|-------|-----|----------------|
| UI/UX | 🟠 medium | Neaktívne tlačidlo nehovorí prečo — `<span>` bez `aria-disabled` aj bez `title` | `resources/views/nova/invoice/buttons-card.blade.php:41` | áno → opravené (`aa67024`) |
| Security | 🟠 medium | Priamy POST na routu vráti neodchytenú výnimku, čiže holú 500 | `app/Http/Controllers/InvoiceActionController.php:21` | áno → opravené (`aa67024`) |
| Reachability | 🟡 low | `Country` je registrovaná obrazovka, ale nevedie na ňu odkaz ani relačný tab | `app/Nova/Country.php` | nie — staršie |
| Performance | 🟡 low | CLS 0,35 na detaile (limit 0,1) — Nova shell, nie táto zmena | `/resources/invoices/{id}` | nie — staršie |

**Severity:** 🔴 high (dáta, peniaze, prístup k cudzím dátam) ・ 🟠 medium (používateľ sa zasekne alebo chyba ostane bez vysvetlenia) ・ 🟡 low (kozmetika, technický dlh) ・ ℹ️ info (len odporúčanie — framework, nie je to chyba a nepočíta sa do výsledku)

#### Odporúčania — framework best practices (Step 6.6.5, len odporúčania)

Nainštalované: Laravel v12.28.1 · PHP ^8.3 · Nova v5.7.2 · Vue 3.5.13 · Tailwind 4.1.4 · browserslist `defaults`

Voliteľné návrhy na budúci refaktoring. Nie sú to chyby, nič som podľa nich
neupravil a neovplyvňujú stav akceptačných kritérií ani výsledok testovania.

| # | Kde | Teraz | Zvážiť pri najbližšom refaktoringu (dostupné v nainštalovanej verzii) | Zdroj |
|---|-----|-------|-------------------------------------------------------------------------|-------|
| 1 | `app/Models/Invoice.php:34` | `match` nad textovými stavmi v accessore | `casts()` + enum cast `InvoiceStatus::class` (Laravel 12) | `laravel.com/docs/12.x/eloquent-mutators#enum-casting` |
| 2 | `resources/js/Components/InvoiceFilter.vue:12` | `props` + `emit('update:modelValue')` | `defineModel()` (Vue 3.5) | `vuejs.org/guide/components/v-model` |

_+1 ďalšie odporúčanie vynechané (limit 5 na task)._

Render this section whenever `framework` ran — with no recommendations it is one
line (*"framework: žiadne odporúčania (Laravel v12.28.1, Vue 3.5.13)"*); with no
docs source reachable it says so instead. When `framework` was not run, it is
the line *"framework: nespustené (--dimensions)"*. A feature newer than the
installed version is never listed here — it is a finding in the table above
(Step 6.6.5, point 4).

#### Negatívne kontroly (Step 6.5.5)

| Kritérium | Čo som vrátil | Test spadol? |
|-----------|---------------|--------------|
| AC-4 | `?? ''` zo dvoch šablón | ✅ áno, 3 testy na `ViewException` |
| AC-6 | `catch (Throwable)` → `catch (DomainException)` | ✅ áno, oba testy dávky |

A row with **"nespadol"** means the criterion is downgraded to ⚠️ partial. Say
it in the table rather than hiding it in prose.

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

`Stav testovania` is derived from the acceptance-criteria table alone. Review
findings sit next to it without changing it, and the framework recommendations
are not even counted as findings — a task with five ℹ️ recommendations and every
criterion ✅ is `✅ PASSED`.

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
- Framework recommendations: 4 ℹ️ (advisory — not counted in any status above)
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

### 9.3 Tick the acceptance criteria that were met

If `config.test_skill.tick_acceptance_criteria == true` (default) and the user
did not pass `--tick-ac=false`, flip `- [ ]` to `- [x]` in the Teamwork
description for every criterion that ended **`✅ verified`**.

Tick **only** `✅`. A `⚠️ partial`, `📋 manual`, `📝 proposed` or `❌ failed`
criterion stays unticked — a tick is a claim that somebody proved it, and a
checklist that lies is worse than one nobody filled in.

> **BYTE-EXACT SAFETY CONTRACT (critical).** The description is the task's only
> copy of a lot of things, and some of them are invisible in a diff of the
> rendered view. In particular **inline images are not attachments** — a pasted
> screenshot lives *only* as a link inside the description markdown, so a rewrite
> that drops that link deletes the screenshot permanently, with no copy in the
> files tab to restore from. Regenerating the description from a parsed model is
> therefore forbidden. The only legal edit is a character-level substitution of
> the six characters `- [ ]` inside the acceptance-criteria block.
>
> Build the new text, then **prove** it is minimal before sending. Refuse to
> write if any assertion fails.

Hand the description to Python through a file, byte-exact. `jq -j` writes the
string without the trailing newline that `jq -r` would add, and the here-string
keeps zsh's `echo` away from it — an `echo` would turn every `\n` escape in the
JSON into a raw newline before `jq` even sees it, and the length assertion
below would then compare against a corrupted original:

```bash
# $BODY = a fresh GET /projects/api/v3/tasks/${TASK_ID}.json taken right before
# the write (Step 3 pattern, HTTP status checked).
jq -j '.task.description' <<<"$BODY" > "/tmp/tw_desc_orig_${TASK_ID}.md"
```

```python
import re
# exactly as the API returned it; newline='' keeps any \r\n untouched
raw = open(f'/tmp/tw_desc_orig_{TASK_ID}.md', encoding='utf-8', newline='').read()

m = re.search(r'(##\s*Akceptačné kritériá\s*\n)(.*?)(\n---)', raw, re.S)   # or the EN heading
assert m, "acceptance-criteria block not found — do not write"

head, block, tail = m.group(1), m.group(2), m.group(3)

# Tick only the lines whose criterion ended ✅. Keep line order and text intact.
new_block = block
for line in VERIFIED_CRITERION_LINES:
    new_block = new_block.replace(f'- [ ] {line}', f'- [x] {line}')

new_raw = raw[:m.start()] + head + new_block + tail + raw[m.end():]

# --- assertions that make the write safe -------------------------------------
assert len(new_raw) == len(raw)                                  # ' ' -> 'x', nothing else
assert new_raw.count('- [x]') == raw.count('- [x]') + N_TICKED
for marker in INLINE_ASSET_MARKERS:                              # e.g. 'tw-inlineimages'
    assert new_raw.count(marker) == raw.count(marker)
assert new_raw.replace('- [x]', '- [ ]') == raw.replace('- [x]', '- [ ]')   # nothing but boxes moved

# --- the payload the PATCH below sends, written only after every assertion held
import json
with open('/tmp/tw_patch_desc.json', 'w', encoding='utf-8') as f:
    json.dump({'task': {'description': new_raw}}, f, ensure_ascii=False)
```

The last assertion is the strongest one and the only one worth remembering: with
every checkbox normalised back, the two texts must be **identical**. If they are
not, something other than a checkbox changed and the write must not happen.

Then send it, and verify the round trip rather than trusting the 200:

```bash
curl -sS -u "$AUTH" -H "Content-Type: application/json" -H "Accept: application/json" \
  -X PATCH --data-binary @/tmp/tw_patch_desc.json \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}.json"
```

Re-fetch the task and confirm the stored description equals what was sent, the
tick count is right, and every inline-asset marker survived. Extract both sides
the same byte-exact way and compare them with `cmp` — never through `echo`:

```bash
# $BODY = the re-fetched task (Step 3 pattern, HTTP status checked).
jq -j '.task.description' /tmp/tw_patch_desc.json > "/tmp/tw_desc_sent_${TASK_ID}.md"
jq -j '.task.description' <<<"$BODY"             > "/tmp/tw_desc_stored_${TASK_ID}.md"
cmp "/tmp/tw_desc_sent_${TASK_ID}.md" "/tmp/tw_desc_stored_${TASK_ID}.md" \
  && echo "round trip OK" || echo "⚠ stored description differs from what was sent" >&2
```

Teamwork's WYSIWYG
has been known to rewrite parts of a submitted body, so "HTTP 200" is not
evidence that what you sent is what is now stored.

On any failed assertion or a mismatched round trip: **do not retry with a
different body.** Report it, leave the description alone, and print the ticks the
user should apply by hand.

Idempotent by construction — a second run finds the boxes already `- [x]`,
substitutes nothing, and the length assertion passes trivially.

### 9.4 Board moves (optional, off by default)

If `config.test_skill.move_on_pass` is set to a non-empty stage name **and** every AC for the task ended ✅, post the task to that stage (reuse the workflow stage resolution from the `teamwork-task` plugin Step 3.3). Same for `move_on_fail` if any AC ended ❌. By default both are empty strings → the skill never touches the board.

The workflow lookup (`GET /projects/api/v3/workflows.json?projectIds=<id>&include=stages`)
needs the task's project id. Take it from `.projectId // .tasklist.meta.projectId`
(Step 3) — v3 task objects have no top-level `projectId`, so reading `.projectId`
alone yields `null`, the lookup finds no workflow, and the move silently does
nothing. If the id is still empty, print a `⚠` and skip the move rather than
guessing a stage.

---

## Blocker handling

Stop and ask via **AskUserQuestion** only on these genuine blockers:
- URL cannot be parsed.
- API returns 401 (token invalid / expired).
- The task has no acceptance criteria *and* `suggest_missing_acceptance_criteria == false` — ask whether to skip the task or enable suggestion.
- The dev server is needed for visual verification but is not running — ask whether to switch to manual scenarios or to skip the criterion.
- The chosen test runner is not installed in the project (e.g. Cypress imports but `npx cypress` is missing) — ask whether to skip or install.
- **A negative control left the working tree dirty** and the restore did not take
  (Step 6.5.5 step 4). Stop: a QA pass must not hand back a repository with a fix
  reverted in it. Show `git status` and the backup path, and let the user restore.
- **The description-tick safety contract failed** (Step 9.3) — one of the byte-exact
  assertions did not hold, or the round trip came back different from what was sent.
  Do not retry with a different body; report it and print the ticks to apply by hand.

Everything else degrades silently with a single-line warning to stderr and continues:
- Comment / time log POST failures.
- Single test execution failures (an `❌ failed` AC is a *result*, not a blocker).
- Browser MCP transient errors — record the criterion as `📋 manual` and move on.
- Attachment download failures.
- A comments fetch that fails on both the v3 and the v1 endpoint (Step 3). The
  task is still tested, but its report says *"comments unavailable"* — an empty
  comment list and a failed request must never read the same.
- A review dimension that cannot run (no browser session for the frontend half of
  performance, an unknown framework for the reachability screen scan, no lock
  file or no docs source for `framework`). Say in the report which dimension was
  skipped and why — a silently skipped dimension reads as a clean bill of health,
  which is the one thing it must never do.

The advisory `framework` dimension is never a blocker and never a reason to ask:
a recommendation waits for a future refactor task, it does not hold up this one.

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
      "evidence": [{ "method": "pest", "file": "tests/Feature/InvoiceListTest.php", "test_name": "it shows invoice list", "duration_ms": 480, "negative_control": "confirmed" }]
    },
    …
  ],
  "review_findings": [
    {
      "dimension": "reachability",
      "severity": "low",
      "title": "Country screen has no menu link and no relation tab",
      "location": "app/Nova/Country.php",
      "detail": "Registered and loads at /resources/countries with 250 rows, but nothing links to it.",
      "in_this_diff": false,
      "fixed": false
    },
    {
      "dimension": "framework",
      "severity": "info",
      "advisory": true,
      "title": "defineModel() instead of props + update:modelValue",
      "location": "resources/js/Components/InvoiceFilter.vue:12",
      "installed_version": "vue 3.5.13",
      "suggestion": "const model = defineModel()",
      "docs": "https://vuejs.org/guide/components/v-model",
      "in_this_diff": true,
      "fixed": false
    }
  ],
  "framework_versions": { "laravel/framework": "v12.28.1", "php": "^8.3", "vue": "3.5.13" },
  "framework_recommendations_dropped": 0,
  "dimensions_run": ["ui_ux", "performance", "security", "reachability", "framework"],
  "dimensions_skipped": []
}
```

`dimension` is one of `ui_ux`, `performance`, `security`, `reachability` or
`framework`. The first four are findings; `framework` items are recommendations
(`"severity": "info"`, `"advisory": true`, `"fixed": false`) and must be left out
of any pass / fail roll-up — `status` above never reflects them.

`dimensions_run` / `dimensions_skipped` are what make an empty
`review_findings` array meaningful: without them, "no findings" and "nobody
looked" are indistinguishable to whatever reads this file next.

This makes the report grep-friendly and lets CI / other tooling pick it up.

Add `${REPORT_DIR}` to `.gitignore` automatically using the same idempotent
pattern as the `teamwork-task` plugin (`/.teamwork-task-test/`).

---

## Security

- The API token lives only in `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (chmod 600).
- Never `echo` the token, never paste it into the report, never write it to `ac.json` / `report.md`.
- Pass auth to `curl` via `-u "$TOKEN:xxx"`, never via the URL query string.
- Browser screenshots may contain confidential customer data — they live under the auto-`.gitignore`d `${REPORT_DIR}` so they cannot accidentally be committed.
