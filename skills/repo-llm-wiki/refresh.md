# refresh — regenerate wiki from current HEAD

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

If a profile arg was passed (`/repo-llm-wiki refresh iac-aws`), set `PROFILE_OVERRIDE = iac-aws`. Valid values: `iac-aws`, `generic`. If invalid, stop with the SKILL.md usage block.

`refresh` is one-shot regeneration. Phase 1 does not attempt incremental updates — every page is regenerated. The only difference from `init` is the guard direction and the log entry behaviour.

---

## Guards

**Guard A** — if `wiki/index.md` does **not** exist, **stop**:
> "No wiki found here (`wiki/index.md` missing). Run `/repo-llm-wiki init` to scaffold one first."

**Guard B** — verify we're inside a git working tree:
```bash
git rev-parse --is-inside-work-tree
```
If this returns anything other than `true`, **stop**:
> "repo-llm-wiki requires a git repository."

---

## Steps

Print: *"Refreshing wiki from current HEAD..."*

1. **Capture prior log SHA.** Before writing anything, parse the SHA from the most recent `## ` entry in `wiki/log.md`. Store as `PRIOR_SHA`. If `wiki/log.md` is missing or unparseable, `PRIOR_SHA = unknown`.

2. **Run Steps 1 and 2 from `init.md`** — repo data collection (§1.1–§1.9) and profile detection (§2). Print the detected profile.

3. **Compute staleness for index page**:
   ```bash
   git rev-list --count <PRIOR_SHA>..HEAD     # → COMMITS_AHEAD
   ```
   If the call fails (PRIOR_SHA unknown or invalid), set `COMMITS_AHEAD = 0` and proceed.

4. **Regenerate every wiki page** in the order specified in `init.md` §4.1–§4.9. **Overwrite** existing pages (do not merge prose). The skip conditions documented in `init.md` apply identically (e.g. skip `wiki/infra.md` if `PROFILE != "iac-aws"`).

   Two exceptions:
   - **`wiki/log.md`** — do **not** overwrite. Prepend a new entry at the top (after the `# Generation log` heading and intro paragraph). The new entry uses the template substitutions from `init.md` §4.8 with `{{COMMAND}}` = `refresh`. Existing entries below are preserved verbatim.
   - **`wiki/.archive/`** — leave untouched.

   **Note on stale module pages**: refresh regenerates module pages but does not delete pages from prior runs. If grouping changed (e.g. a Lambda was added and the cluster reshaped), old `wiki/modules/<old-slug>.md` files may remain. `lint` E5 will flag any orphan as a missing index link; the user removes orphans manually. Phase 1 accepts this; auto-pruning is a future improvement.

5. **`wiki/index.md` staleness suffix**: set `STALE_SUFFIX` empty. After a successful refresh, the new log entry's SHA equals HEAD, so the index is current at the moment this run completes. The `COMMITS_AHEAD` figure computed in step 3 is shown in the completion summary (informational only). The staleness warning in `index.md` is meaningful only between refreshes; `lint` (W1) handles that case.

6. **`AGENTS.md` block**: if the block (`<!-- repo-llm-wiki: begin -->`) already exists, leave the file untouched and print: `AGENTS.md wiki block already present`. Otherwise append the block per `init.md` §4.10. **Never** modify existing block content during `refresh` — Phase 1 treats the block as immutable once present.

---

## Completion summary

```
Wiki refreshed (<PAGES_WRITTEN_COUNT> pages)

  <list of pages actually written this run>

Profile: <PROFILE> [<FLAVOR>]
SHA: <REPO_SHA[0:7]>  Branch: <REPO_BRANCH>
Previous SHA: <PRIOR_SHA[0:7]>  (<COMMITS_AHEAD> commits since)

Run /repo-llm-wiki lint to validate.
```

If `PRIOR_SHA == REPO_SHA` (refresh on an unchanged repo), the line "Previous SHA …" should still be printed; `COMMITS_AHEAD == 0` is informative.
