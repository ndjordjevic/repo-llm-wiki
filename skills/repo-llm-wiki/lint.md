# lint — validate wiki health

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

`lint` is non-destructive: it reads only. It collects all errors and warnings and prints a grouped report at the end. **Run all checks even if early checks fail.**

The v0.2 wiki has no source-file citations and no mermaid diagrams; lint reflects that.

---

## Guards

**Guard A** — if `wiki/index.md` does **not** exist, **stop**:
> "No wiki found here (`wiki/index.md` missing). Run `/repo-llm-wiki init` first."

---

## Errors (each makes the wiki invalid)

Record `<page>: <problem>` in an `ERRORS` list.

### E1. Broken wikilinks
For every `[[<slug>]]` occurrence in any file under `wiki/`, check `wiki/<slug>.md` exists. Slugs may contain a slash (`lambdas/amendArrangement`). Record: `<source-page>: broken wikilink [[<slug>]]`.

### E2. Source-file citations present
The v0.2 wiki must not contain relative-path links to source files. For every Markdown link in `wiki/` whose target matches `^\.\./` or `^\./` (relative path leaving `wiki/`), record: `<source-page>: source-file citation should be removed: <target>`.

This is a stricter inversion of the v0.1 rule: file links used to be required, now they're forbidden.

### E3. Missing log entry
`wiki/log.md` must contain at least one line starting with `## `. If absent: `wiki/log.md: no generation entries`.

### E4. Missing AGENTS.md block
`AGENTS.md` must exist at the repo root and contain `<!-- repo-llm-wiki: begin -->`. Otherwise: `AGENTS.md: missing repo-llm-wiki block`.

### E5. index.md must link every top-level page
For every `*.md` file directly under `wiki/` (excluding `log.md` and anything inside `.archive/`), check that `wiki/index.md` references it via `[[<slug>]]`. Record: `wiki/index.md: missing link to <slug>`.

### E6. Category–detail consistency
For every detail page under `wiki/<category>/<slug>.md` (where `<category>` ∈ {`lambdas`, `cli`, `migrations`, `infra`}), check:
- `wiki/<category>.md` exists, AND
- `wiki/<category>.md` references the detail page via `[[<category>/<slug>]]`.

For every `[[<category>/<slug>]]` link inside `wiki/<category>.md`, check the detail page exists.

Record orphans both ways:
- `wiki/<category>.md: missing link to <slug>` (detail page exists, list page doesn't link)
- `wiki/<category>/<slug>.md: orphan (no listing in <category>.md)` (link missing)
- `wiki/<category>.md: dangling link [[<category>/<slug>]] (page missing)`

### E7. Empty pages
For every `*.md` file under `wiki/`, check it has at least 50 non-whitespace characters of body (excluding the H1). Record: `<page>: page is effectively empty`. Empty pages indicate a generation failure.

---

## Warnings (don't fail the lint)

Record in `WARNINGS`.

### W1. Stale wiki
Parse the SHA from the most recent `## ` entry in `wiki/log.md` (look for `- SHA: <hex>` on the line below). Run:
```bash
git rev-list --count <log-sha>..HEAD     # → COMMITS_AHEAD
```
If `COMMITS_AHEAD > 20`, record: `wiki staleness: HEAD is <N> commits ahead of last refresh`. If the SHA can't be parsed or `git rev-list` fails: `wiki staleness: cannot compare with HEAD (log SHA <X> not found)`.

### W2. Confidence: low markers
For each page containing `Confidence: low`, record: `<page>: contains "Confidence: low" — verify`.

### W3. Suspiciously short detail page
For each `wiki/<category>/<slug>.md` whose body is < 200 characters (excluding the H1 and the `> Type:` line), record: `<page>: detail page is very short — flow narrative may be missing`.

---

## Report

```
repo-llm-wiki lint

ERRORS (<n>)
  - <page>: <problem>
  - ...
  (none)

WARNINGS (<n>)
  - <page>: <problem>
  - ...
  (none)

Result: <PASS | FAIL>
```

`PASS` if `ERRORS` is empty; warnings do not fail.

Do not auto-fix. Print the report and stop.
