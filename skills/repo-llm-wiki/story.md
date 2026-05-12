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

Exclude wiki-internal files: drop any path starting with `wiki/`.

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

Build a `PATH→PAGES` index by scanning every `*.md` file under `wiki/` (exclude `wiki/log.md` and anything under `wiki/.archive/`) for file references. A file reference is any of:
- A markdown link whose target does not start with `[[` and contains a `/` (i.e. a path-like string)
- A backtick-quoted token containing a `/` that looks like a path (e.g. `` `cmd/amendArrangement/main.go` ``)
- A Mermaid node label containing a path fragment

For each changed file in `CHANGED_FILES`, look it up in the index (exact match, then longest-prefix match on directory).

**Fallback routing** for files that match no wiki page. Phase 1's category set is closed: only `lambdas`, `cli`, `packages`, `scripts`, `infra`. Route by file path:

| Changed file path | Routes to |
|---|---|
| `cmd/<X>/...`, `app/cmd/<X>/...` (under a dir that has a detail page) | `wiki/lambdas/<X>.md` or `wiki/cli/<X>.md` (whichever detail page exists) |
| `cmd/<X>/...`, `app/cmd/<X>/...` (no detail page exists) | `wiki/lambdas.md` (will be flagged unmatched if subagent can't place it; new entrypoints need `refresh`) |
| `infra/**/*.tf`, `infra/**/*.tfvars` | `wiki/infra.md` (and any matching `wiki/infra/<slug>.md` per citations) |
| `*.sh` at repo root, `Makefile`, `.github/workflows/**` | `wiki/scripts.md` |
| Top-level Go package directories with citations on a packages page | `wiki/packages/<slug>.md` |
| Anything else | log as unmatched in the completion summary; do not invent a routing target |

**Story does not create new top-level pages.** If the dev's branch adds a new Lambda entrypoint or a new top-level package, story will route its files to the closest existing page if possible and flag them as unmatched otherwise. New detail pages require a `refresh` run. Note this in the completion summary when unmatched > 0 so the dev knows.

**Large-page-set guard:**
If the resulting `AFFECTED_PAGES` set has more than 6 pages, print:
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
4. This instruction:

   > Update this wiki page to reflect the code changes shown in the diff. Keep the existing format, headings, and wikilink structure. Rewrite only sections that are affected by the diff. Do **not** add relative paths to source files (no `../app/...`, no `./infra/...`). Refer to handlers, services, and resources by their conceptual role, not by file path. Allowed `[[wikilinks]]` targets are: `index`, `log`, the five category pages (`lambdas`, `cli`, `packages`, `scripts`, `infra`), and existing detail pages under those categories (e.g. `lambdas/foo`, `infra/dynamodb-tables`). Remove any wikilink whose target you cannot verify from the page's existing links. Return the full updated page content as Markdown — nothing else.

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
Unmatched files: <count>  (not covered by any wiki page)

Run /repo-llm-wiki lint to validate.
```

If `unmatched files > 0`, list the unmatched paths below the summary so the dev can spot-check.

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
