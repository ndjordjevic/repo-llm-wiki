# repo-llm-wiki

A Claude Code skill that generates a committable Markdown wiki from the repo it lives in.

## What it does

Run `/repo-llm-wiki init` and the skill reads your repo — source layout, go.mod / package.json / terraform files, DynamoDB schema, GitHub workflows, domain model — then writes a `wiki/` directory with cross-linked pages you can commit and browse in any Markdown viewer.

Version numbers, GSI names, Lambda counts, and workflow files come from the source files themselves, not from README prose. The result is accurate in a way that generated docs usually aren't.

## Output

```
wiki/
  index.md                   entry point, links to everything
  overview.md                plain-language summary, tech stack
  architecture.md            component diagram (Mermaid) + data flow
  runbook.md                 prerequisites, build, test, deploy, workflows
  infra.md                   AWS resources, DynamoDB tables/GSIs, environments (when Terraform files present)
  glossary.md                domain terms drawn from model files
  repo-map.md                directory tree with callouts for loose files, deprecated paths, sensitive names
  modules/<slug>.md          one page per logical Lambda group (3–8 groups)
  log.md                     append-only generation history
AGENTS.md                    wiki pointer block appended for other agents
```

## Commands

| Command | What it does |
|---|---|
| `/repo-llm-wiki init` | First-time generation. Stops if `wiki/index.md` already exists. |
| `/repo-llm-wiki refresh` | Regenerate all pages from HEAD. Requires an existing wiki. |
| `/repo-llm-wiki lint` | Validate wiki health: broken wikilinks, stale citations, missing log entries. |

`infra.md` is generated automatically when Terraform files are found in `infra/` — no configuration needed.

## Skill layout

```
skills/repo-llm-wiki/
  SKILL.md          dispatch and git policy
  init.md           full init procedure
  refresh.md        regen procedure (delegates to init.md §1–4)
  lint.md           E1–E6 errors, W1–W4 warnings
  templates/
    wiki/
      index.md.tmpl
      log.md.tmpl
```
