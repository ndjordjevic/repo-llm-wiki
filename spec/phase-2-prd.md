# repo-llm-wiki — Phase 2 PRD: Living wiki via `story`

> **Status:** draft · **Date:** 2026-05-12 · **Author:** nenad

## 1. Problem

Phase 1 produces a snapshot wiki. The moment a dev lands a change, it starts drifting. The research doc (`research/00-research-notes.md` §7.1) calls drift the single biggest risk to Phase 1 surviving past a 6-month half-life.

A full `refresh` is the wrong shape for everyday work: it regenerates every page (token-expensive, non-deterministic, orphans pages), ignores the *why* a dev knows in their head, and doesn't get run.

## 2. Goal

A diff-aware update path the dev runs themselves at the natural moment — when they finish a story. Cheap, narrow, and captures intent the diff alone can't carry.

## 3. Scope (Phase 2.0)

**In:**

- New subcommand `/repo-llm-wiki story "<description>"`.
- Reads `git status` + `git diff` (working tree + staged + optional base-branch compare).
- Identifies which wiki pages are affected by the touched files.
- Updates only those pages (overwrites the affected sections, not whole files).
- Prepends an entry to `wiki/log.md` capturing the dev description + brief auto-summary of changes.

**Out (deferred):**

- GitHub Action / post-merge automation (Phase 2.1).
- PR-title / Jira mining beyond the dev-supplied description (Phase 3).
- Multi-repo propagation (Phase 4).
- Wiki-PR generation. Dev commits manually; matches Phase 1's git policy.

## 4. User flow

```
$ /repo-llm-wiki story "Added SQS retry queue for failed auth callbacks"
Inspecting changes (origin/main..HEAD + working tree, 12 files)...
Affected wiki pages: lambdas/authCallback.md, lambdas.md, infra/sqs-queues.md
Updated 3 pages. Logged story to wiki/log.md.
Run /repo-llm-wiki lint to validate.
```

Dev reviews the diff, commits the wiki update alongside (or after) their code change.

## 5. Requirements

### 5.1 Inputs

- **Required:** description string (positional arg, quoted). Reject empty / <10 chars with usage message.
- **Implicit:** current git working tree state.

### 5.2 Change detection

Default range is the **full feature branch** — all commits since the branch diverged from its base, plus any uncommitted local changes:

1. **Auto-detect base branch.** Try `main`, `master`, `develop` in order (first one found in `git branch -r`). If none found, fall back to `--since` flag.
2. **Find fork point.** Use `git merge-base HEAD <base>` — not a plain `<base>..HEAD` — so the diff stays correct even if the base branch has advanced since branching.
3. **Combine ranges.** `git diff <fork-point>..HEAD` (committed changes) + `git diff HEAD` (uncommitted). Deduplicate the file list.

Flag `--since <ref>` overrides auto-detection for non-standard setups (e.g. branched off a release branch, squash-merge workflow).

Stop if no changes detected in either range. Print: *"No changes to summarise. Nothing to update."*

### 5.3 Page-mapping rule

Each wiki page declares its "source paths" — the file globs whose changes should trigger an update. Phase 2 uses a **citation-derived map**: scan each wiki page for file references (already required by Phase 1's citation discipline), build path→pages index, intersect with changed paths.

Fallback when a touched file matches no page: route by directory prefix to the closest existing detail page (`wiki/<category>/<slug>.md`) or, failing that, to its parent category list (`wiki/lambdas.md`, `wiki/cli.md`, `wiki/packages.md`, `wiki/scripts.md`, `wiki/infra.md`). Files that don't match any of these are logged as **unmatched** in the run output so the dev sees what wasn't covered. Story does not invent new pages or categories — Phase 1's category set is closed; new top-level entries require `refresh`.

### 5.4 Per-page update

- Affected pages are regenerated **in place** by the same subagent pattern as `init` §3 — but each subagent receives only the page's prior content, the relevant diff hunks, and the dev description, and returns a full new page body.
- Citations must be re-validated (no dangling file refs).
- `wiki/index.md` is updated only if a page was added or removed (rare in Phase 2).

### 5.5 Log entry

Prepended to `wiki/log.md` after the heading:

```
## {{TIMESTAMP}} — story
- Description: {{DEV_DESCRIPTION}}
- SHA: {{SHA}} (or "working tree" if uncommitted)
- Branch: {{BRANCH}}
- Files changed: {{FILE_COUNT}} ({{LINES_ADDED}}+/{{LINES_REMOVED}}-)
- Pages updated: {{PAGE_LIST}}
- Summary: {{AUTO_SUMMARY}}      ← 1–2 sentences, LLM-generated from diff
- Tool: repo-llm-wiki v{{TOOL_VERSION}}
```

`AUTO_SUMMARY` is a short factual restatement of the diff (the *what*), distinct from `DEV_DESCRIPTION` (the *why*).

### 5.6 Guards

- Not in a git working tree → stop.
- `wiki/index.md` missing → stop, instruct user to run `init`.
- Same-SHA + clean working tree → stop with "no changes" message.
- Description missing/too short → print usage.

### 5.7 Git policy

Inherits Phase 1's canonical rule (`SKILL.md`): never commit or push. `story` writes to `wiki/` and `wiki/log.md` only; the dev commits.

## 6. Non-goals (explicit)

- No attempt to mine commit messages for description text — dev supplies it.
- No reverse direction (wiki edit → code suggestion).
- No conflict resolution if two devs run `story` on overlapping pages — last writer wins; lint catches downstream.

## 7. Success criteria

- A dev who runs `story` after a feature lands gets <30s wall time and ≤3 pages touched for a typical change.
- The resulting log entries are skim-readable: a new joiner can scroll `wiki/log.md` and understand the project's last 20 changes without opening any other page.
- `lint` passes after every `story` run on a healthy wiki.

## 8. Open questions

- **Auto-trigger.** A git `post-commit` hook could call `story` non-interactively, using commit subject as description. Defer to Phase 2.1 — needs UX validation that hook-mode doesn't annoy devs.
- **Section-level updates.** Phase 2.0 regenerates whole affected pages. If pages get long, section-anchored updates may be needed; not yet.
- **Diff size cap.** Very large stories (e.g. dependency bumps touching 500 files) should probably suggest `refresh` instead. Define threshold in implementation.

