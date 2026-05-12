# repo-llm-wiki — Phase 2 implementation plan

> Pairs with `phase-2-prd.md`. Targets a working `story` subcommand on top of the Phase 1 skill layout.

## 1. Files to add / change

```
skills/repo-llm-wiki/
  SKILL.md           ← register `story` in dispatch + subcommand table
  story.md           ← NEW: full procedure
  init.md            ← (no changes; story references its §3 subagent pattern)
  refresh.md         ← (no changes)
  lint.md            ← add rule for malformed `story` log entries
  templates/wiki/
    log.md.tmpl      ← add `story` entry template block
```

No template changes to page files — story regenerates existing pages in-place.

## 2. `story.md` outline

**Note (2026-05-12):** the design landed differently from the original sketch below. The shipped `story.md` is structured as:

- **Top-of-file `⛔ Forbidden behaviours` + `✅ Required actions for NEW_ENTRIES` callouts** — these proved necessary after the model repeatedly defaulted to a "pending row + run refresh later" pattern. Front-loading the checklist made the agent comply.
- **Step 4d (NEW_ENTRIES)** creates fresh detail pages inline (NEW-DETAIL subagent → write `wiki/<category>/<name>.md` from `init.md` §5.2 template). It does **not** suppress list-page updates and defer to `refresh` as originally sketched.
- **Verification block** (mechanical shell checks for detail-page existence, wikilink presence, count parity) runs before the completion summary. Without this, the agent sometimes skipped index updates or count bumps.

The procedural outline below documents the original intent; the shipped procedure refines it as above.

Mirror the structure of `refresh.md` (small, procedural). Sections:

1. **Trigger & args.** Parse the description string. Reject if absent / <10 chars. Parse optional `--since <ref>`.
2. **Guards.** Git working tree check → `wiki/index.md` exists → changes present.
3. **Inspect changes.**

   **Step 3a — find the base ref:**
   ```bash
   # Auto-detect: first of main / master / develop present in remote refs
   git branch -r | grep -E 'origin/(main|master|develop)' | head -1
   # → BASE_BRANCH (e.g. "origin/main"). If nothing found, require --since.
   ```
   `--since <ref>` overrides `BASE_BRANCH` directly.

   **Step 3b — find the fork point:**
   ```bash
   git merge-base HEAD <BASE_BRANCH>   # → FORK_SHA
   ```
   If `merge-base` fails (shallow clone, no common ancestor), stop:
   > "Could not determine branch base. Pass `--since <ref>` explicitly."

   **Step 3c — collect changes:**
   ```bash
   git diff --name-only <FORK_SHA>..HEAD        # committed branch changes
   git diff --name-only HEAD                    # uncommitted changes
   git diff --stat <FORK_SHA>..HEAD             # line counts for committed
   git diff --stat HEAD                         # line counts for uncommitted
   git diff <FORK_SHA>..HEAD -- <files>         # full hunks (per-file, later)
   git diff HEAD -- <files>                     # full hunks for uncommitted
   ```
   Merge the two file lists (deduplicate). Build `CHANGED_FILES` with paths + line deltas (sum both ranges).

   Print: `Inspecting changes (<N> files across <FORK_SHA[0:7]>..<HEAD_SHA[0:7]> + working tree)...`
4. **Map to wiki pages.**
   - Scan every `wiki/**/*.md` (excluding `log.md`, `.archive/`) for citations — markdown links, backticks containing `/`, mermaid file labels.
   - Build `PATH→PAGES` index. Intersect with `CHANGED_FILES`.
   - For unmatched files: route by closest directory prefix to `modules/<slug>.md`; else `architecture.md`.
   - Resulting set: `AFFECTED_PAGES`.
   - Cap: if `len(AFFECTED_PAGES) > 6` OR `len(CHANGED_FILES) > 100`, print warning and ask the user to confirm or run `/repo-llm-wiki refresh` instead.
5. **Fan-out update (parallel subagents).** One subagent per affected page. Each receives:
   - Page's current content.
   - Diff hunks touching files cited by that page.
   - Dev description.
   - The page's source-paths inventory.
   
   Returns: full new page body + citation list. Same writing style/contract as `init.md` §3.
6. **Write pages.** Overwrite each affected page atomically. Do not touch unaffected pages.
7. **Generate `AUTO_SUMMARY`.** One inline LLM call: input = `git diff --stat` + top-level file list. Output: 1–2 sentence factual summary. Cap at ~280 chars.
8. **Prepend log entry.** Use the new `story` block from `log.md.tmpl`. Insert after the `# Generation log` heading; preserve all prior entries verbatim.
9. **Completion summary.** Mirror `refresh.md`'s completion block:
   ```
   Wiki story logged (<N> pages updated)

     <list of pages>

   Description: <DEV_DESCRIPTION>
   SHA: <SHA[0:7]>  Branch: <BRANCH>
   Files: <N> (<+>/<->)
   Unmatched files: <count>  (see above)

   Run /repo-llm-wiki lint to validate.
   ```

## 3. `SKILL.md` edits

- Subcommand table — add row: `story | Update wiki pages from a finished story`.
- Dispatch list — add: `**`story`** → `story.md``.
- Usage block — add line: `  story "<description>"  — update wiki pages affected by current changes`.

## 4. `lint.md` edits

Add one new check:

- **E7 (error):** `wiki/log.md` story entry missing required fields (`Description`, `SHA`, `Pages updated`, `Summary`).
- **W5 (warning):** story entries older than 30 days that reference pages no longer present in `wiki/`.

## 5. `log.md.tmpl` edits

Append a second template block below the existing one:

```
## {{TIMESTAMP}} — story
- Description: {{DEV_DESCRIPTION}}
- SHA: {{SHA}}
- Branch: {{BRANCH}}
- Files changed: {{FILE_COUNT}} ({{LINES_ADDED}}+/{{LINES_REMOVED}}-)
- Pages updated: {{PAGE_LIST}}
- Summary: {{AUTO_SUMMARY}}
- Tool: repo-llm-wiki v{{TOOL_VERSION}}
```

## 6. Edge cases to handle in `story.md`

| Case | Behaviour |
|---|---|
| No commits on branch yet (FORK_SHA == HEAD) | Only uncommitted changes in scope. Use `git diff HEAD`. Log SHA as `working tree`. |
| Shallow clone / no common ancestor | Stop. Instruct dev to pass `--since <ref>` explicitly. |
| Detached HEAD | Branch field = `(detached at <SHA[0:7]>)`. |
| Only wiki files changed | Stop. *"Story touches only wiki/ — nothing to summarise."* |
| File added that no page references | Route to `repo-map.md` + log as unmatched. |
| File deleted that pages cite | Affected pages list includes them; subagent must rewrite to remove dangling citations. |
| `--since` ref invalid | Stop with the git error. |
| Description contains markdown that could break the log | Escape backticks and `##`. |

## 7. Sequencing

1. Land `story.md` + `SKILL.md` dispatch + `log.md.tmpl` block. (Day 1)
2. Manual test on `tfaws-dregdata-dhs-app-authorisation` with a real recent change. Iterate on prompt fidelity for §5 subagents. (Day 2–3)
3. Add `lint` rules E7 / W5. (Day 4)
4. Doc updates: `README.md` Commands table, `research/` follow-up note recording deviations from PRD. (Day 4)
5. Cut `v0.5.0` of the skill.

## 8. Test plan (manual)

Run against a clone of the target repo:

1. **Single-module story.** Touch one Lambda's handler + tests → `story "added rate limiter"`. Expect: 1–2 pages updated, log entry sane, lint clean.
2. **Cross-cutting story.** Add a Terraform resource + wire it into a Lambda + bump infra docs → expect `infra.md`, `architecture.md`, one module page touched.
3. **Wiki-only change.** Edit `wiki/overview.md` manually → run `story` → expect early-stop.
4. **Large change.** Bump dep across 200 files → expect threshold warning + suggestion to `refresh`.
5. **Description guard.** Run `story ""` → usage. Run `story "fix"` → reject (too short).
6. **`--since main`.** Branch with 5 commits → expect aggregated diff.

## 9. Risks

- **Citation map false negatives.** If a page doesn't cite a file it should cover, that file's change won't route to it. Mitigation: lint already enforces citation discipline (Phase 1); story logs unmatched files for the dev to spot-check.
- **Subagent regenerating a page poorly.** Same risk as `refresh`. Mitigation: page-level diff is reviewable in `git diff wiki/` before the dev commits.
- **Hook-mode foot-gun (later).** If we auto-run `story` from `post-commit`, a noisy commit could pollute the log. Keep it manual in Phase 2.0.

## 10. Definition of done

- `/repo-llm-wiki story "<desc>"` runs end-to-end on the target repo, produces a well-formed log entry, and updates the correct pages.
- `lint` passes immediately after.
- README documents the subcommand.
- One real example commit exists in the target repo showing the wiki diff alongside the code diff.
