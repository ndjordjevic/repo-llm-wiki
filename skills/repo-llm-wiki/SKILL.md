---
name: repo-llm-wiki
description: "Generates a committable Markdown wiki from the repo it lives in — invoke with /repo-llm-wiki"
version: "0.1.0"
last_updated: "2026-05-02"

compatible_agents:
  tested:
    - claude
  untested:
    - cursor
    - copilot

categories:
  - documentation
  - productivity
  - onboarding

job_roles:
  - developer
  - architect

author: "Nenad Djordjevic"
github: "ndjordjevic"
license: "apache-2.0"
trigger: "/repo-llm-wiki"
---

# /repo-llm-wiki

Generates a small, accurate, committable Markdown wiki *inside* a repo, written at the altitude humans and agents actually use. Sibling to `pin-llm-wiki`, which ingests external sources; this one ingests the repo it lives in.

## Trigger phrases

- **`/repo-llm-wiki`** (with subcommands: `init`, `refresh`, `lint`)
- "Build a wiki from this repo"
- "Generate the repo wiki"
- "Refresh the repo-llm-wiki output"
- "Lint the repo wiki"

```
repo source files (Go, TF, README, workflows, ...)
    ↓  collect (shell, file reads)
in-memory facts (versions, GSIs, lambdas, ...)
    ↓  generate
wiki/  (committed, cited, agent-readable)
    ↓  lint
a healthy, queryable knowledge base
```

## Phase 1 subcommands

| Command | Status |
|---|---|
| `init` | Phase 1 |
| `refresh` | Phase 1 |
| `lint` | Phase 1 |

Future phases (not in this skill): live updates from PRs, Jira/Confluence ingestion, wiki-of-wikis.

## Skill directory

This SKILL.md and its sibling files (`init.md`, `refresh.md`, `lint.md`, `templates/...`) live inside the skill directory: `~/.claude/skills/repo-llm-wiki/`, `~/.copilot/skills/repo-llm-wiki/`, `~/.cursor/skills/repo-llm-wiki/`, or the project-local equivalents. In this repository the canonical copy is **`skills/repo-llm-wiki/`**. All `templates/...` paths in this skill are relative to whichever skill directory the loading tool used.

## Execution model

This skill is plain Markdown instructions. The "agent" referenced throughout is the host LLM session running the skill (Claude Code, Cursor, or Copilot). Single-agent: one session reads instructions, runs shell commands, reads files, and writes pages, in order. No external models, no orchestration daemon.

## Dispatch

1. Identify the subcommand from the invocation args (the first word after `/repo-llm-wiki`).
2. Route — read the sibling file in this skill directory and follow its instructions exactly:
   - **`init`** → `init.md`
   - **`refresh`** → `refresh.md`
   - **`lint`** → `lint.md`
4. If the subcommand is missing or unrecognised, print and stop:
   ```
   Usage: /repo-llm-wiki <subcommand>
     init             — scaffold and generate a new wiki
     refresh          — regenerate wiki from current HEAD
     lint             — validate wiki health
   ```
5. Do not proceed beyond this dispatch step before reading the target file.

## Git policy (canonical)

**Never run `git commit` or `git push`** after any subcommand — `init`, `refresh`, `lint`, or any auto-fix — unless the human explicitly asked to commit in this conversation. Subcommand files reference this policy without restating it. The wiki's own `AGENTS.md` block carries the same rule for downstream agents.
