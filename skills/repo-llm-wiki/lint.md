# lint — validate wiki health

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

`lint` is non-destructive: it reads only. It collects all errors and warnings and prints a grouped report at the end. **Run all checks even if early checks fail** — exit only after the full report is printed.

---

## Guards

**Guard A** — if `wiki/index.md` does **not** exist, **stop**:
> "No wiki found here (`wiki/index.md` missing). Run `/repo-llm-wiki init` first."

---

## Errors (each makes the wiki invalid)

For each finding, record `<page>:<problem>` in an `ERRORS` list.

### E1. Broken wikilinks
For every `[[<slug>]]` occurrence in any file under `wiki/`, check that `wiki/<slug>.md` exists. Slugs may contain a slash (`modules/foo`). For each missing target, record: `<source-page>: broken wikilink [[<slug>]]`.

### E2. Broken file citations
For every Markdown link in `wiki/` whose target is a relative path (`./...` or `../...`), resolve from the file's directory and check that the path exists in the repo. For each broken link, record: `<source-page>: broken citation -> <resolved-path>`.

(Skip external links: `http://`, `https://`, `mailto:`, anchors `#...`.)

### E3. Missing log entry
`wiki/log.md` must contain at least one line starting with `## ` (a generation entry). If absent, record: `wiki/log.md: no generation entries`.

### E4. Missing AGENTS.md block
`AGENTS.md` must exist at the repo root and contain `<!-- repo-llm-wiki: begin -->`. If `AGENTS.md` is absent, or the marker is missing, record: `AGENTS.md: missing repo-llm-wiki block`.

### E5. index.md missing links
For every `*.md` file directly under `wiki/` (excluding `log.md` and anything inside `.archive/`), check that `wiki/index.md` references it via `[[<slug>]]`. For each missing entry, record: `wiki/index.md: missing link to <slug>`.

For module pages under `wiki/modules/`, check that `wiki/index.md` references them as `[[modules/<slug>]]`.

### E6. Mermaid blocks
For each ` ```mermaid ` fenced block found in any wiki page:
- The first non-blank line must start with one of: `flowchart`, `graph`, `sequenceDiagram`, `classDiagram`, `stateDiagram`, `erDiagram`, `gantt`, `pie`, `journey`, `mindmap`.
- The block must contain at least one line with `-->`, `---`, `:`, or `==>`.

Failures recorded as: `<page>: malformed mermaid block (no diagram-type keyword)` or `<page>: malformed mermaid block (no edges/nodes)`.

This is regex-only and catches obvious corruption; it does not validate semantics.

---

## Warnings (don't fail the lint, but flag)

For each finding, record in `WARNINGS` list.

### W1. Stale wiki
Parse the SHA from the most recent `## ` entry in `wiki/log.md` (look for `- SHA: <hex>` on the line below). Run:
```bash
git rev-list --count <log-sha>..HEAD     # → COMMITS_AHEAD
```
If the call succeeds and `COMMITS_AHEAD > 20`, record: `wiki staleness: HEAD is <N> commits ahead of last refresh`.

If the SHA can't be parsed or `git rev-list` fails (e.g. log SHA is no longer in history after a force-push), record a warning: `wiki staleness: cannot compare with HEAD (log SHA <X> not found)`.

### W2. Uncited pages
For each file matching `wiki/architecture.md` or `wiki/modules/*.md`, count occurrences of `([` (opening of a Markdown link). If the count is **zero**, record: `<page>: no citations`.

(This is a deterministic per-page rule. Per-paragraph quality requires human judgment and is out of scope for lint.)

### W3. Draft mermaid present
If `wiki/architecture.md` exists and contains `<!-- draft -->`, record: `wiki/architecture.md: mermaid diagram marked draft — verify arrows before sharing externally`.

### W4. Confidence: low markers
For each page in `wiki/` containing `Confidence: low`, record: `<page>: contains "Confidence: low" — verify`.

---

## Report

Print the report exactly in this format. Sections with no findings are still printed with `(none)`.

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

`Result: PASS` if `ERRORS` is empty (warnings do not fail).
`Result: FAIL` otherwise.

Do not auto-fix anything. Print the report and stop.
