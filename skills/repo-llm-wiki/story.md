# story — update wiki pages from a finished dev story

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

`story` is a diff-aware update. It updates pages affected by the current branch's changes, and **for every new top-level entry it creates the fresh detail page inline**.

---

## ⛔ Forbidden behaviours (read this first)

If you find yourself about to do any of the following, STOP — you are misreading the instructions:

1. **Never insert a "detail page pending — run /repo-llm-wiki refresh" row** into a list page. Story creates the detail page itself; there is no pending state.
2. **Never insert a row into a list page without a `[[wikilink]]` target.** Every row in `wiki/lambdas.md`, `wiki/cli.md`, `wiki/packages.md` is a `[[wikilink]]` to a detail page that exists on disk.
3. **Never tell the dev to "run /repo-llm-wiki refresh" to finish the work.** Story is self-contained for new entries. If you are tempted to write that line in the completion summary, you skipped Step 5.
4. **Never bump a count on one page (e.g. `wiki/lambdas.md` headline → 24) without also bumping it on `wiki/index.md`, `wiki/infra.md`, and `wiki/infra/lambdas.md`.** Counts must be consistent across all four.

## ✅ Required actions for NEW_ENTRIES (must all happen in one run)

When the branch adds a brand-new `cmd/<X>/` or `app/cmd/<X>/` (call it a `NEW_ENTRY`), you MUST do all of the following before printing the completion summary:

1. Spawn a NEW-DETAIL subagent (Step 5) that reads `main.go`, the handler, and one service file for `<X>`, returning a flow narrative.
2. Write `wiki/lambdas/<X>.md` (or `wiki/cli/<X>.md`) using the detail-page template from `init.md` §5.2. **The file must exist on disk before Step 7 starts.**
3. Include `wiki/lambdas.md` (or `cli.md`) in `AFFECTED_PAGES`. Its subagent inserts a `[[lambdas/<X>]]` wikilink row and bumps the headline count.
4. Include `wiki/infra/lambdas.md` in `AFFECTED_PAGES`. Its subagent adds the Terraform-module-name bullet and bumps the count.
5. Include `wiki/infra.md` in `AFFECTED_PAGES`. Its subagent bumps the Lambda count in `## Resource types`.
6. Include `wiki/index.md` in `AFFECTED_PAGES`. Its subagent bumps the count in `## Categories` AND in the overview paragraph.
7. Run the verification block (end of Step 9) before printing the completion summary.

---

## Trigger & args

Invoked as: `/repo-llm-wiki story "<description>"` with an optional `--since <ref>` flag.

- **`<description>`** — required. The dev's one-sentence "why" for this story. Must be ≥ 10 characters. If missing or too short, stop:
  ```
  Usage: /repo-llm-wiki story "<description>" [--since <ref>]
    description    required — what this story does / why (≥ 10 chars)
    --since <ref>  override base branch auto-detection with a specific ref
  ```

- **`--since <ref>`** — optional. Overrides base-branch auto-detection. Useful when the branch was cut from a release branch or a non-standard base.

---

## Guards

Run all three before any work.

**Guard A** — verify git working tree:
```bash
git rev-parse --is-inside-work-tree
```
If anything other than `true`, stop:
> "repo-llm-wiki requires a git repository."

**Guard B** — verify wiki exists:
```bash
test -f wiki/index.md
```
If missing, stop:
> "No wiki found here (`wiki/index.md` missing). Run `/repo-llm-wiki init` first."

**Guard C** — verify description:
If description is absent or < 10 chars, print usage (above) and stop.

---

## Step 1 — Collect repo identity

```bash
git rev-parse HEAD                          # → HEAD_SHA
git rev-parse --abbrev-ref HEAD             # → BRANCH
```

---

## Step 2 — Detect base branch and find fork point

**Step 2a — Auto-detect base branch** (skip if `--since` was supplied):

```bash
git branch -r | grep -E 'origin/(main|master|develop)$' | head -1 | xargs
```

Keep the full `origin/<branch>` ref as `BASE_REF` — do **not** strip `origin/`. The dev's local clone may not have a tracking branch for `main` (common on feature-branch workflows), so `git merge-base HEAD main` can fail while `git merge-base HEAD origin/main` succeeds.

If nothing found and `--since` was not supplied, stop:
> "Could not auto-detect a base branch (tried origin/main, origin/master, origin/develop). Pass `--since <ref>` explicitly."

If `--since <ref>` was supplied, set `BASE_REF = <ref>` directly and skip the auto-detection above.

**Step 2b — Find the fork point:**

```bash
git merge-base HEAD <BASE_REF>   # → FORK_SHA
```

If `merge-base` fails (shallow clone, no common ancestor, or the ref doesn't exist locally), stop:
> "Could not determine branch base with `git merge-base <BASE_REF>`. Run `git fetch` first, or pass `--since <ref>` explicitly."

---

## Step 3 — Collect changes

Print: *"Inspecting changes (`<FORK_SHA[0:7]>`..<HEAD_SHA[0:7]> + working tree)..."*

**Step 3a — File list:**
```bash
git diff --name-only <FORK_SHA>..HEAD    # committed branch changes
git diff --name-only HEAD                 # uncommitted staged/unstaged changes
```
Merge and deduplicate → `CHANGED_FILES`.

**Exclude wiki-tooling files** (these are not source code; they're the wiki's own scaffolding):
- Any path starting with `wiki/`
- `AGENTS.md` at the repo root (written by `init`)
- `CLAUDE.md`, `.cursor/`, `.copilot/`, `.claude/` (host-agent config)

These exclusions apply both to the change-mapping pass and to the "unmatched" report — don't print them as unmatched, they're handled.

**Edge cases:**
- If `FORK_SHA == HEAD_SHA`: only uncommitted changes are in scope. Use `git diff --name-only HEAD` only. Log SHA as `working tree`.
- If `CHANGED_FILES` is empty after exclusion, stop:
  > "No changes to summarise. Nothing to update."
- If the only changes are inside `wiki/`: stop:
  > "Story touches only wiki/ — nothing to summarise."

**Step 3b — Line counts:**
```bash
git diff --stat <FORK_SHA>..HEAD     # for committed range
git diff --stat HEAD                  # for uncommitted
```
Sum: `LINES_ADDED`, `LINES_REMOVED`, `FILE_COUNT = len(CHANGED_FILES)`.

**Step 3c — Large-change guard:**
If `FILE_COUNT > 100`, print a warning and ask the user to confirm before continuing:
> "This story touches <FILE_COUNT> files. That's large for `story` — consider running `/repo-llm-wiki refresh` instead. Proceed anyway? (yes/no)"

---

## Step 4 — Map changed files to wiki pages

Phase 1 wiki pages deliberately contain **no source-file citations** (see `init.md` design intent). So the routing here is driven primarily by a **slug-derived reverse map** built from the wiki's existing structure, with text-citation scanning as a secondary signal.

**Step 4a — Build the slug-derived reverse map.**

Enumerate existing detail pages, then claim source paths from each slug:

```bash
ls wiki/lambdas/*.md 2>/dev/null   # → LAMBDA_SLUGS (strip dir + .md extension)
ls wiki/cli/*.md     2>/dev/null   # → CLI_SLUGS
ls wiki/packages/*.md 2>/dev/null  # → PACKAGE_SLUGS
ls wiki/infra/*.md   2>/dev/null   # → INFRA_SLUGS
```

For each slug `<S>` in `LAMBDA_SLUGS`, the page `wiki/lambdas/<S>.md` claims:
- `cmd/<S>/**`, `app/cmd/<S>/**`
- `internal/handler/<S>/**`, `app/internal/handler/<S>/**`
- `internal/service/<S>/**`, `app/internal/service/<S>/**`

Same pattern for `CLI_SLUGS` → `wiki/cli/<S>.md`.

For each slug `<S>` in `PACKAGE_SLUGS`, the page `wiki/packages/<S>.md` claims:
- `<S>/**` (top-level package dir)

For each slug `<S>` in `INFRA_SLUGS`, the page `wiki/infra/<S>.md` claims the resource-type `.tf` file when the slug matches conventionally:
- `wiki/infra/lambdas.md` claims `infra/lambda.tf`, `infra/api-gateway.tf`, `infra/locals.tf` (lambda wiring)
- `wiki/infra/dynamodb-tables.md` claims `infra/dynamodb.tf`
- generally `wiki/infra/<S>.md` claims `infra/<slug-singular>.tf` (best-effort name match)

**Step 4b — Build the text-citation map (secondary).**

Scan every `*.md` file under `wiki/` (exclude `wiki/log.md` and anything under `wiki/.archive/`) for explicit file references:
- A backtick-quoted token containing a `/` that looks like a path (e.g. `` `infra/dynamodb.tf` ``)
- Markdown links whose target contains a `/` and an extension

Add these to the `PATH→PAGES` index built in 4a.

**Step 4c — Route each changed file.**

For each file in `CHANGED_FILES`:
1. Exact match in the index → assign to that page set.
2. Longest-prefix directory match in the index → assign.
3. Fallback table below.
4. None of the above → unmatched.

**Fallback table** (consulted only after 4a/4b miss):

| Changed file path | Routes to |
|---|---|
| `cmd/<X>/**`, `app/cmd/<X>/**` where no `wiki/lambdas/<X>.md` or `wiki/cli/<X>.md` exists | **NEW ENTRY** — see Step 4d below |
| `app/internal/handler/<X>/**` or `app/internal/service/<X>/**` where no `wiki/lambdas/<X>.md` exists | NEW ENTRY (handler/service for an unwritten Lambda) — see 4d |
| `app/internal/model/**`, `app/internal/mapper/**`, `app/internal/mock/**`, other shared `internal/` paths | log as **shared-internal** (expected unmatched; no single page owns these) |
| `infra/**/*.tf`, `infra/**/*.tfvars` not claimed by 4a | `wiki/infra.md` |
| `infra/swaggers/**`, `infra/openapi/**`, `*.openapi.yaml` | `wiki/infra.md` (API Gateway shape) |
| `*.sh` at repo root, `Makefile`, `.github/workflows/**` | `wiki/scripts.md` |
| Anything else | unmatched |

**Step 4d — New top-level entries.**

If `CHANGED_FILES` adds a brand-new `cmd/<X>/`, `app/cmd/<X>/`, or top-level package dir with **no matching detail page** in the wiki, treat it as a `NEW_ENTRY`. Story creates the detail page and threads it through all dependent pages in the same run — `refresh` is not required after.

For every `NEW_ENTRY`, record:
- `name` — cmd-dir name (e.g. `latestAuthorisations`)
- `category` — `lambdas` | `cli` | `packages`
- `source_root` — e.g. `app/cmd/latestAuthorisations/`
- `infra_module_name` (Lambdas only) — derive from the Terraform diff (`identifier = "..."` field) or kebab-case the name as fallback

Then schedule the following work (executed in Steps 5 and 7):

| Action | Page | Role |
|---|---|---|
| Spawn NEW-DETAIL subagent → write detail page | `wiki/<category>/<name>.md` | created in Step 5 |
| Insert wikilink row + bump headline count | `wiki/<category>.md` (list page) | `list-page-with-adds` |
| Add bullet + bump count (Lambdas only) | `wiki/infra/lambdas.md` | `infra-inventory` |
| Bump Lambda count (Lambdas only) | `wiki/infra.md` | `infra-inventory` |
| Bump Categories count + overview prose count | `wiki/index.md` | `index-with-count-bump` |

For CLI / package NEW_ENTRIES: no infra-side pages — only the detail page, list page, and `wiki/index.md` are touched.

For non-Lambda infra changes (new DynamoDB GSI on an existing table, new SQS queue, environment additions): no NEW_ENTRY — route via the fallback table to existing infra pages.

**Large-page-set guard.** If `AFFECTED_PAGES` exceeds 6 pages, print and ask to confirm:
> "This story affects <N> wiki pages. That's broad for `story` — consider `/repo-llm-wiki refresh` instead. Proceed anyway? (yes/no)"

---

## Step 5 — Read sources for NEW_ENTRIES (NEW-DETAIL subagents)

Skip this step if `NEW_ENTRIES` is empty.

For each `NEW_ENTRY`, spawn a **NEW-DETAIL subagent** (in parallel with all other NEW-DETAIL subagents — single message, multiple `Agent` tool calls). Each subagent uses the same prompt shape as `init.md` §3 **Subagent A** (cmd/ flow narratives) or §3 **Subagent E** (packages), depending on category.

**For Lambda / CLI NEW_ENTRIES — Subagent A-style prompt:**

Inputs to pass:
- `name` (e.g. `latestAuthorisations`)
- `source_root` (e.g. `app/cmd/latestAuthorisations/`)
- The dev description (helps disambiguate intent when source is ambiguous)
- Up to 200 lines each of: `<source_root>/main.go`, the relevant `app/internal/handler/<name>/handler.go` if it exists, and the first non-test `.go` file under the handler's imported service package

The subagent must:
1. Classify type: `Lambda` if `main.go` imports `aws-lambda-go` or calls `lambda.Start(`, else `CLI tool`.
2. Infer trigger using the Subagent A trigger rules from `init.md` §3 Subagent A.
3. Produce a 3-6 sentence flow narrative (handler → service → data path → response).
4. Return as the structured JSON-ish object format from `init.md` §3 Subagent A.

**For package NEW_ENTRIES — Subagent E-style prompt:**

Inputs: the package directory, module path. Subagent reads first non-test `.go` file, extracts exports, writes summary per `init.md` §3 Subagent E.

**Main agent action after NEW-DETAIL returns:**

Using the returned classification + flow narrative, write the new detail page directly using the templates from `init.md` §5.2 (Lambda/CLI detail page contract). Path:
- `wiki/lambdas/<name>.md` for Lambda
- `wiki/cli/<name>.md` for CLI tool
- `wiki/packages/<name>.md` for package

Print: `created: wiki/<category>/<name>.md`.

**Important:** complete this write BEFORE spawning Step 6's list-page subagents. The list-page subagent's prompt will reference the new detail page by slug, and we want the file to exist on disk when lint runs after.

---

## Step 6 — Collect diff hunks for existing affected pages

For each entry in `AFFECTED_PAGES` (excluding pages already written in Step 5), identify its cited / claimed source files (from the routing in Step 4). Fetch full diff hunks for only those files:

```bash
git diff <FORK_SHA>..HEAD -- <cited-files>   # committed
git diff HEAD -- <cited-files>               # uncommitted
```

For list-page roles with new entries, also include each NEW_ENTRY's `{name, category, trigger, one_line_summary}` (from Step 5's returns) so the subagent can add the wikilink row.

---

## Step 7 — Fan-out update (parallel subagents)

Spawn one subagent per affected page **in a single message** (parallel), using the `Agent` tool with `subagent_type=general-purpose`.

Each subagent prompt must be self-contained (no view of this conversation). Include:

1. The page's **current full content**.
2. The **diff hunks** for files cited by this page (from Step 6).
3. The **dev description** verbatim.
4. The page's **role** — see below.
5. Role-specific instructions (below).
6. For list-page-with-adds and index-with-count-bump roles: the `NEW_ENTRIES` payload (names, categories, triggers, 1-line summaries) — so the subagent has everything needed to insert rows / bump counts.

**Determine the page's role:**

| Role | Pages | When |
|---|---|---|
| `list-page` | `wiki/lambdas.md`, `wiki/cli.md`, `wiki/packages.md` | No new entries for that category; existing rows may be edited |
| `list-page-with-adds` | same | One or more NEW_ENTRIES belong to that category |
| `infra-inventory` | `wiki/infra.md`, `wiki/infra/<slug>.md` | Always when in `AFFECTED_PAGES` |
| `detail-page` | `wiki/lambdas/<X>.md`, `wiki/cli/<X>.md`, `wiki/packages/<X>.md` | Existing detail page in `AFFECTED_PAGES` (story is updating, not creating) |
| `prose` | `wiki/scripts.md` | When scripts/workflows changed |
| `index-with-count-bump` | `wiki/index.md` | One or more NEW_ENTRIES exist; only update the affected category's count in `## Categories` |

**Base instruction (all pages):**

> Update this wiki page to reflect the code changes shown in the diff. Keep the existing format, headings, and wikilink structure. Rewrite only sections that are affected by the diff. Do **not** add relative paths to source files (no `../app/...`, no `./infra/...`). Refer to handlers, services, and resources by their conceptual role, not by file path. Return the full updated page content as Markdown — nothing else.

**Additional instruction for `list-page` (no new entries)** (append to base):

> Every data row in this page's table is a `[[wikilink]]` to an existing detail page. Do not add new rows. You may update the one-line description column for rows whose detail page appears in the diff. Do not change the headline count.

**Additional instruction for `list-page-with-adds`** (append to base):

> **There are new entries to add to this table.** For each entry in the NEW_ENTRIES payload that belongs to this category, insert a new row in the table in alphabetical position, using this exact format:
>
> `| [[<category>/<name>]] | <trigger> | <one_line_summary> |`
>
> (For CLI tools, the column header is `Invocation` instead of `Trigger`; use `CLI invocation` as the value.) After inserting, **find every count reference on this page and bump it** by the number of new entries you added. Count references include the headline line (e.g. "This repo deploys 23 AWS Lambda functions" → "24"), any subsection introductions, and any "There are N ..." sentences. You may also update the description column for *existing* rows whose detail page appears in the diff. Do not remove or reorder existing rows except to maintain alphabetical order around the new insertion.

**Additional instruction for `infra-inventory`** (append to base):

> This page is a flat resource inventory. For `wiki/infra/lambdas.md` (or similar resource-type pages), add a bullet for each new Terraform module in the diff (use the kebab-case module name, e.g. `latest-authorisations`), maintaining the original ordering. Then **find every count reference on this page and bump it** by the number of new resources added — including the page's opening sentence ("There are 23 Lambda functions ..." → "24"). For `wiki/infra.md` specifically, bump counts in: the `## Resource types` section bullets (`[[infra/lambdas]] — N Lambda functions`), the `## Small inventories` table if relevant, and any prose count references in the overview.

**Additional instruction for `detail-page`** (append to base):

> This page describes a single component. Update its prose to reflect the diff (e.g. a service gaining a new method, a handler gaining a new endpoint). Keep the page 3-6 sentences. Do not split into subsections.

**Additional instruction for `index-with-count-bump`** (append to base):

> Two passes:
> 1. Find the `## Categories` section. For each category in NEW_ENTRIES, update its bullet to reflect the new count (e.g. `- [[lambdas]] — 23 Lambda functions` → `- [[lambdas]] — 24 Lambda functions`).
> 2. Find the overview / introduction paragraph (typically the first prose paragraph after the H1 metadata line). If it states counts like "deployed as 23 AWS Lambda functions, with 2 companion CLI tools" or "N Lambda functions" or "N CLI tools", bump those numbers to the new totals as well.
>
> Leave all other content (tech stack, tests, meta) untouched. Do not rewrite the overview prose — only edit the specific numbers.

**Additional instruction for `prose`** (append to base):

> Update the relevant subsection(s) to reflect what changed. Keep the rest untouched.

Subagent returns the full new page body. Write it directly, overwriting the prior content.

---

## Step 8 — Generate AUTO_SUMMARY

One inline LLM call (not a subagent). Input:
- The output of `git diff --stat <FORK_SHA>..HEAD`
- `CHANGED_FILES` list
- Dev description

Prompt: *"In 1–2 sentences (max 280 chars), describe what changed in the code — the what, not the why. Be factual and specific about which components or areas changed."*

→ `AUTO_SUMMARY`

---

## Step 9 — Prepend log entry

Read `wiki/log.md`. After the `# Generation log` heading (and any blank line), insert:

```
## <TIMESTAMP> — story
- Description: <DEV_DESCRIPTION>
- SHA: <HEAD_SHA[0:7]> (or "working tree" if FORK_SHA == HEAD_SHA)
- Branch: <BRANCH>
- Files changed: <FILE_COUNT> (<LINES_ADDED>+/<LINES_REMOVED>-)
- Pages updated: <comma-separated list of updated page paths>
- Summary: <AUTO_SUMMARY>
- Tool: repo-llm-wiki v0.5.0
```

`TIMESTAMP` format: `YYYY-MM-DD HH:MM UTC`.

Escape any `##` or backtick characters in `DEV_DESCRIPTION` to avoid breaking the Markdown structure.

Preserve all prior entries below verbatim.

---

## Verification block (run before printing completion summary)

If `NEW_ENTRIES` is non-empty, execute the following checks from the repo root. Any failure means you skipped a required action — STOP, complete it, and re-run.

```bash
HALT=0

# Per-entry: detail page exists; parent list page links to it
for ENTRY in <name1:category1> <name2:category2> ...; do
  NAME=${ENTRY%:*}
  CAT=${ENTRY#*:}
  test -f "wiki/$CAT/$NAME.md" \
    || { echo "❌ detail page wiki/$CAT/$NAME.md missing"; HALT=1; }
  grep -q "\[\[$CAT/$NAME\]\]" "wiki/$CAT.md" 2>/dev/null \
    || { echo "❌ wiki/$CAT.md does not link [[$CAT/$NAME]]"; HALT=1; }
done

# Count parity per affected category (only when category has list+infra+index counts)
for CAT in lambdas cli packages; do
  test -f "wiki/$CAT.md" || continue
  ROWS=$(grep -c "^| \[\[$CAT/" "wiki/$CAT.md")
  IDX=$(grep -oE "\[\[$CAT\]\] — [0-9]+" wiki/index.md | grep -oE '[0-9]+')
  [ -n "$IDX" ] && [ "$ROWS" != "$IDX" ] \
    && { echo "❌ wiki/$CAT.md has $ROWS rows but wiki/index.md Categories says $IDX"; HALT=1; }
  # Infra parity only for Lambdas (E9-governed)
  if [ "$CAT" = "lambdas" ] && [ -f wiki/infra/lambdas.md ]; then
    INFRA=$(grep -c '^- ' wiki/infra/lambdas.md)
    [ "$ROWS" != "$INFRA" ] \
      && { echo "❌ wiki/lambdas.md has $ROWS rows but wiki/infra/lambdas.md has $INFRA bullets"; HALT=1; }
  fi
done

[ "$HALT" = "1" ] && echo "VERIFICATION FAILED — fix before printing completion summary" && exit 1
[ "$HALT" = "0" ] && echo "✓ verification passed"
```

If any check fails, do NOT proceed to the completion summary. Identify the missing action (see the ✅ checklist at the top of this file), complete it, and re-run.

---

## Completion summary

```
Wiki story logged (<P> pages updated, <D> detail pages created)

  Updated:
    <list of pages updated, one per line>

  Created:
    <list of new detail pages, one per line>

Description: <DEV_DESCRIPTION>
SHA: <HEAD_SHA[0:7]>  Branch: <BRANCH>
Files: <FILE_COUNT> (<LINES_ADDED>+/<LINES_REMOVED>-)
Shared internal: <count>  (expected — no single page owns these)
Unmatched files: <count>  (not covered by any wiki page)

Run /repo-llm-wiki lint to validate.
```

If `unmatched > 0` or `shared-internal > 0`, list those paths below the summary so the dev can spot-check.

If `NEW_ENTRIES` is non-empty, the "Created" block above already lists them. Do not print an extra warning — the wiki is now consistent with the code, and `refresh` is not required.

---

## Edge cases reference

| Case | Behaviour |
|---|---|
| `FORK_SHA == HEAD_SHA` (no commits on branch yet) | Use uncommitted diff only. Log SHA as `working tree`. |
| Shallow clone / `merge-base` fails / `--since` ref not found | Stop. Surface the git error and suggest `git fetch` or pass `--since <ref>` explicitly. |
| Detached HEAD | `BRANCH = (detached at <HEAD_SHA[0:7]>)`. Proceed. |
| Only wiki files changed | Stop. *"Story touches only wiki/ — nothing to summarise."* |
| File deleted that pages cite | Subagent for that page receives the deletion diff and must rewrite to remove dangling citations. |
| Description contains line-start `##` or pipe characters | Replace line-start `##` with `\#\#` and pipes with `\|` so the Markdown bullet stays well-formed. Inline backticks are fine. |
| NEW_ENTRY has no `app/internal/handler/<name>/` directory | NEW-DETAIL subagent reads `main.go` only and produces a best-effort flow narrative. If the page lands under 200 chars (lint W3), it's still written — the dev can run `refresh` for a richer pass. |
| Subagent output contains a broken wikilink | Write the page anyway; the verification block + `lint` will catch it. |
