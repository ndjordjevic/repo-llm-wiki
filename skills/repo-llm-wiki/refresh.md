# refresh — regenerate wiki from current HEAD

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

`refresh` is one-shot regeneration — every page is regenerated. The only differences from `init` are the guard direction, the log entry behaviour, and the orphan-cleanup pass.

---

## Guards

**Guard A** — if `wiki/index.md` does **not** exist, **stop**:
> "No wiki found here (`wiki/index.md` missing). Run `/repo-llm-wiki init` to scaffold one first."

**Guard B** — verify we're inside a git working tree:
```bash
git rev-parse --is-inside-work-tree
```
If anything other than `true`, **stop**:
> "repo-llm-wiki requires a git repository."

---

## Steps

Print: *"Refreshing wiki from current HEAD..."*

1. **Capture prior log SHA.** Parse the SHA from the most recent `## ` entry in `wiki/log.md`. Store as `PRIOR_SHA`. If missing/unparseable, `PRIOR_SHA = unknown`.

2. **Snapshot existing pages.** Record the set of every `*.md` file currently under `wiki/` (excluding `log.md` and `.archive/`). Call this `PRIOR_PAGES`. Used in step 5 for orphan cleanup.

3. **Run Steps 1–5 from `init.md`** — repo data collection, subagent fan-out, layout decision, directory creation, page writing. All skip conditions and the infra-split rule apply identically. **Overwrite** existing pages (do not merge prose).

   Two exceptions:
   - **`wiki/log.md`** — do **not** overwrite. Prepend a new entry at the top (after the `# Generation log` heading) using the template substitutions from `init.md` §5.4 with `{{COMMAND}}` = `refresh`. Existing entries below are preserved verbatim.
   - **`AGENTS.md` block**: if the block already exists, leave the file untouched and print: `AGENTS.md wiki block already present`. Otherwise append per `init.md` §5.6. **Never modify** existing block content.

4. **Compute staleness for completion summary**:
   ```bash
   git rev-list --count <PRIOR_SHA>..HEAD     # → COMMITS_AHEAD
   ```
   If `PRIOR_SHA == unknown` or the call fails, `COMMITS_AHEAD = 0`.

5. **Orphan cleanup.** Compute `WRITTEN_PAGES` = pages written in step 3. For every page in `PRIOR_PAGES` that is **not** in `WRITTEN_PAGES`, the layout changed (a lambda was removed, a category went empty, an infra split flipped). Move it to `wiki/.archive/<original-path>` rather than deleting, so the user can recover prose if a regeneration regressed it.

   ```bash
   mkdir -p wiki/.archive
   # for each orphan: mv "wiki/<path>" "wiki/.archive/<path>"
   ```

   Print one line per moved page: `archived: wiki/<path>`.

---

## Completion summary

```
Wiki refreshed (<PAGES_WRITTEN_COUNT> pages, <ORPHANS_COUNT> archived)

  <list of pages actually written this run>

SHA: <REPO_SHA[0:7]>  Branch: <REPO_BRANCH>
Previous SHA: <PRIOR_SHA[0:7]>  (<COMMITS_AHEAD> commits since)

Run /repo-llm-wiki lint to validate.
```

If `PRIOR_SHA == REPO_SHA`, still print the "Previous SHA" line; `COMMITS_AHEAD == 0` is informative.
