---
name: repo-llm-wiki
description: "Generates a committable Markdown wiki from the repo it lives in — invoke with /repo-llm-wiki"
version: "0.3.0"
last_updated: "2026-05-11"

compatible_agents:
  tested:
    - claude
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

Generates a small, accurate, committable Markdown wiki *inside* a repo, written at the altitude humans and agents actually use.

## Trigger phrases

- **`/repo-llm-wiki`** (with subcommands: `init`, `refresh`, `lint`)
- "Build a wiki from this repo"
- "Refresh the repo-llm-wiki output"
- "Lint the repo wiki"

```
repo source files (Go, TF, README, workflows, scripts, ...)
    ↓  enumerate (cheap shell + reads — main agent)
inventory (cmd dirs, tf resources, scripts, versions)
    ↓  deep analyse in parallel (subagents)
flow narratives + resource summaries + overview prose
    ↓  decide layout + write pages (main agent)
wiki/  (committed, drill-down, agent-readable)
    ↓  lint
a healthy, queryable knowledge base
```

## Subcommands

| Command | Description |
|---|---|
| `init` | Scaffold and generate a new wiki |
| `refresh` | Regenerate wiki from current HEAD |
| `lint` | Validate wiki health |

## Skill directory

This SKILL.md and its sibling files (`init.md`, `refresh.md`, `lint.md`, `templates/...`) live inside the skill directory: `~/.claude/skills/repo-llm-wiki/`, `~/.copilot/skills/repo-llm-wiki/`, `~/.cursor/skills/repo-llm-wiki/`, or the project-local equivalents. In this repository the canonical copy is **`skills/repo-llm-wiki/`**. All `templates/...` paths in this skill are relative to whichever skill directory the loading tool used.

## Execution model

This skill is plain Markdown instructions. The "agent" referenced throughout is the host LLM session running the skill (Claude Code, Cursor, or Copilot).

**Two-layer**: the main session does the cheap deterministic work (shell enumeration, layout decisions, page writing) and **fans out to subagents in parallel** for the expensive analysis pass — reading every Lambda's entrypoint chain, classifying Terraform resources, grouping scripts, drafting the overview paragraph. Subagent returns are short, structured payloads the main agent assembles into pages. This keeps style consistent (one writer) and protects the main context (subagent context is discarded after each task).

No external models, no orchestration daemon. On hosts without a subagent primitive, the main agent performs the analysis itself inline; output will be the same but slower and noisier.

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
