# story — update wiki pages from a finished dev story

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

`story` is a diff-aware, narrow update. It updates only the wiki pages affected by the current branch's changes — it does not regenerate the whole wiki.

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
- `cmd/<S>/**`
- `app/cmd/<S>/**`
- `app/internal/handler/<S>/**`
- `app/internal/service/<S>/**`
- `internal/handler/<S>/**`, `internal/service/<S>/**`

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

**Step 4d — New top-level entries (HARD RULE).**

If `CHANGED_FILES` adds a brand-new `cmd/<X>/`, `app/cmd/<X>/`, top-level package dir, or any new entrypoint with **no matching detail page** in the wiki, treat it as a `NEW_ENTRY`. Phase 1's category list pages are wikilink-only tables, and adding a non-wikilink row breaks lint E9 (count parity vs `infra/<resource>.md`).

For every `NEW_ENTRY`:

1. **Discard all files under that entry from routing.** Do not add `cmd/<X>/**`, `app/cmd/<X>/**`, `app/internal/handler/<X>/**`, `app/internal/service/<X>/**` to any page's diff-hunk set.
2. **Do not add `wiki/lambdas.md` / `wiki/cli.md` / `wiki/packages.md` to `AFFECTED_PAGES` solely because of a `NEW_ENTRY`.** A list page is only in `AFFECTED_PAGES` when an *existing* row's detail page has changed.
3. **Record** the entry under `NEW_ENTRIES` with its kind (Lambda / CLI / package) and source path.
4. The completion summary surfaces a loud "run `/repo-llm-wiki refresh` to generate the new detail page" notice.

**Reasoning for the main agent:** a new Lambda does *not* by itself justify updating `wiki/lambdas.md`. The list page is a wikilink table; the row can only be added once the detail page exists, and only `refresh` creates detail pages. Story is read-only on the list page until a `refresh` runs.

The Terraform side (`infra/<resource>.md`) is the *one exception*: those are flat inventory bullets, and adding `latest-authorisations` to `infra/lambdas.md` is fine because it's a deployed resource name, not a wikilink.

**Large-page-set guard.** If `AFFECTED_PAGES` (after applying 4d) exceeds 6 pages, print:
> "This story affects <N> wiki pages. That's broad for `story` — consider running `/repo-llm-wiki refresh` instead. Proceed anyway? (yes/no)"

---

## Step 5 — Collect full diff hunks for affected pages

For each affected page, identify its cited files (from the scan in step 4). Fetch full diff hunks for only those files:

```bash
git diff <FORK_SHA>..HEAD -- <cited-files>   # committed
git diff HEAD -- <cited-files>               # uncommitted
```

These hunks are passed to the subagent for that page.

---

## Step 6 — Fan-out update (parallel subagents)

Spawn one subagent per affected page **in a single message** (parallel), using the `Agent` tool with `subagent_type=general-purpose`.

Each subagent prompt must be self-contained (no view of this conversation). Include:

1. The page's **current full content**.
2. The **diff hunks** for files cited by this page (from step 5).
3. The **dev description** verbatim.
4. The page's **role** (list page vs detail page vs infra inventory) — see below.
5. Role-specific instructions.

**Determine the page's role:**

- **List page**: `wiki/lambdas.md`, `wiki/cli.md`, `wiki/packages.md` — flat wikilink tables, one row per existing detail page.
- **Infra inventory**: `wiki/infra.md`, `wiki/infra/<slug>.md` — Markdown tables or bulleted resource lists; adding bullets for new resources is fine (no wikilink requirement).
- **Detail page**: `wiki/lambdas/<X>.md`, `wiki/cli/<X>.md`, `wiki/packages/<X>.md`, `wiki/scripts.md`, `wiki/index.md` — prose pages.

**Base instruction (all pages):**

> Update this wiki page to reflect the code changes shown in the diff. Keep the existing format, headings, and wikilink structure. Rewrite only sections that are affected by the diff. Do **not** add relative paths to source files (no `../app/...`, no `./infra/...`). Refer to handlers, services, and resources by their conceptual role, not by file path. Allowed `[[wikilinks]]` targets are: `index`, `log`, the five category pages (`lambdas`, `cli`, `packages`, `scripts`, `infra`), and existing detail pages under those categories (e.g. `lambdas/foo`, `infra/dynamodb-tables`). Remove any wikilink whose target you cannot verify from the page's existing links. Return the full updated page content as Markdown — nothing else.

**Additional instruction for LIST pages** (append to base):

> **CRITICAL: this is a category list page.** Every data row in this page's table is a `[[wikilink]]` to an existing detail page. You **MUST NOT add new rows** to this table, even if the diff mentions a new Lambda, CLI tool, or package — the new entry has no detail page yet, and adding a non-wikilink row breaks lint validation (E9: count parity with `infra/<resource>.md`). If you see a new entry in the diff, **leave the table unchanged**. You may only modify existing rows whose detail page appears in the diff (e.g. rewriting the one-line description). Do not change the headline count (e.g. "This repo deploys N AWS Lambda functions") — that number stays as-is until `refresh` runs.

**Additional instruction for INFRA INVENTORY pages** (append to base):

> This page is a flat resource inventory. Adding bullets / table rows for new deployed resources (e.g. a new Terraform module) is fine — the inventory does not require wikilinks. Keep the alphabetical or original ordering of the existing list.

**Additional instruction for DETAIL pages** (append to base):

> This page describes a single component. Update its prose to reflect the diff (e.g. a service gaining a new method, a handler gaining a new endpoint). Keep the page 3-6 sentences. Do not split into subsections.

Subagent returns the full new page body. Write it directly, overwriting the prior content.

**`wiki/index.md`**: update only if a page was added or removed in this run (rare). Otherwise leave untouched.

---

## Step 7 — Generate AUTO_SUMMARY

One inline LLM call (not a subagent). Input:
- The output of `git diff --stat <FORK_SHA>..HEAD`
- `CHANGED_FILES` list
- Dev description

Prompt: *"In 1–2 sentences (max 280 chars), describe what changed in the code — the what, not the why. Be factual and specific about which components or areas changed."*

→ `AUTO_SUMMARY`

---

## Step 8 — Prepend log entry

Read `wiki/log.md`. After the `# Generation log` heading (and any blank line), insert:

```
## <TIMESTAMP> — story
- Description: <DEV_DESCRIPTION>
- SHA: <HEAD_SHA[0:7]> (or "working tree" if FORK_SHA == HEAD_SHA)
- Branch: <BRANCH>
- Files changed: <FILE_COUNT> (<LINES_ADDED>+/<LINES_REMOVED>-)
- Pages updated: <comma-separated list of updated page paths>
- Summary: <AUTO_SUMMARY>
- Tool: repo-llm-wiki v<TOOL_VERSION>
```

`TIMESTAMP` format: `YYYY-MM-DD HH:MM UTC`.

Escape any `##` or backtick characters in `DEV_DESCRIPTION` to avoid breaking the Markdown structure.

Preserve all prior entries below verbatim.

---

## Completion summary

```
Wiki story logged (<N> pages updated)

  <list of wiki pages updated this run, one per line>

Description: <DEV_DESCRIPTION>
SHA: <HEAD_SHA[0:7]>  Branch: <BRANCH>
Files: <FILE_COUNT> (<LINES_ADDED>+/<LINES_REMOVED>-)
Shared internal: <count>  (expected — no single page owns these)
Unmatched files: <count>  (not covered by any wiki page)
New top-level entries: <count>

Run /repo-llm-wiki lint to validate.
```

If `unmatched > 0` or `shared-internal > 0`, list those paths below the summary so the dev can spot-check.

If `NEW_ENTRIES` is non-empty, append a loud block:

```
⚠ NEW TOP-LEVEL ENTRIES DETECTED

These additions need a structural wiki update — story does not create new detail pages or list-page rows:

  - <entry name> (category: lambdas | cli | packages) — <source path>

Run /repo-llm-wiki refresh to generate the missing detail pages and update the category list. Lint may report E1/E9/E5 until refresh runs.
```

---

## Edge cases reference

| Case | Behaviour |
|---|---|
| No commits on branch yet (`FORK_SHA == HEAD_SHA`) | Use uncommitted diff only. Log SHA as `working tree`. |
| Shallow clone / `merge-base` fails | Stop. Instruct dev to pass `--since <ref>`. |
| Detached HEAD | `BRANCH = (detached at <HEAD_SHA[0:7]>)`. Proceed. |
| Only wiki files changed | Stop. *"Story touches only wiki/ — nothing to summarise."* |
| File added that no page references | Try the fallback routing in Step 4; otherwise log as unmatched. Story does not create new top-level pages — those require `refresh`. |
| File deleted that pages cite | Subagent for that page receives the deletion diff and must rewrite to remove dangling citations. |
| `--since` ref invalid or not found | Let git error surface. Stop with the git error message. |
| Description contains line-start `##` or pipe characters | Replace line-start `##` with `\#\#` and pipes with `\|` so the Markdown bullet stays well-formed. Inline backticks in the description are fine — leave them. |
| Subagent returns a page with a broken wikilink | Write the page anyway; lint will catch it. Do not block the run. |
| Branch adds a new Lambda / package not yet on any wiki page | Files routed to the parent category list page (`lambdas.md` etc) where possible; flagged unmatched otherwise. Tell the dev in the completion summary to run `refresh` for new top-level entries. |
