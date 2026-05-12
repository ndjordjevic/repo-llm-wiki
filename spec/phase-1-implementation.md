# repo-llm-wiki — Phase 1 Implementation Plan

> Companion to `phase-1-prd.md`. Covers *how* the skill is built: file layout, step-by-step agent instructions, generation order, token strategy, and lint rules.

Status: draft 4 · Date: 2026-05-12

> **Draft-3 supersession notice.** After running v0.1 on the validation repo the output was rejected as too heavy (architecture/repo-map/glossary pages, broken file-link citations in Obsidian, mermaid diagrams nobody could read, lambdas lumped into module clusters). v0.2 pivots to a **drill-down** wiki: `index.md` → category page → per-item flow narrative. The **authoritative source of the v0.2 instructions is `skills/repo-llm-wiki/{init,refresh,lint}.md`** — the SKILL files are now canonical and have been rewritten end-to-end.
>
> This document keeps the v0.1 detail for historical context and to preserve the parts of the design that carry through unchanged (repo data collection in §3, profile-detection ideas in §4). The page-generation sections (§5.x) describe the old layout and are **superseded** wholesale — read `init.md` instead.
>
> Carries through: §1 skill file layout (same files), §3 data collection (commands unchanged), §2 SKILL.md dispatch (same), git policy (same). Superseded: §4 profile detection (replaced wholesale by category discovery — v0.2 does not classify repos into profiles), §5 page generation, §6 profile-specific emphasis, the mermaid rules in §5.6, the module-grouping algorithm in §5.3, every page contract that references file-link citations.

## 0. Execution model

The skill is plain Markdown instructions. The "agent" referenced throughout is the **host LLM session** running the skill (Claude Code, Cursor, or Copilot) — there is no external model invocation, no separate process, no orchestration daemon.

**v0.4 is multi-agent for analysis, single-writer for pages.** The main session runs the cheap deterministic shell pass (§3 below) and the page-writing pass. Between those, it fans out to **up to five subagents in parallel**: Subagent A (cmd/ flow narratives, sharded into batches of ≤15 when >20 dirs), B (Terraform resources), C (scripts grouping), D (overview paragraph), E (package introspection for go-library/mixed repos). Subagent prompts are self-contained; their returns are short structured payloads. See `init.md` Step 2 for the contracts. Hosts without a subagent primitive fall back to inline analysis — same output, slower, noisier main context.

---

## 1. Skill file layout

Mirrors the `pin-llm-wiki` structure. Every file is plain Markdown — the skill is just instructions to the agent.

```
skills/repo-llm-wiki/
  SKILL.md              # entry point: metadata, dispatch table, git policy
  init.md               # /repo-llm-wiki init
  refresh.md            # /repo-llm-wiki refresh
  lint.md               # /repo-llm-wiki lint
  templates/
    AGENTS_BLOCK.md     # block appended to (or inserted in) AGENTS.md
    wiki/
      index.md.tmpl     # index page template (TOC shell)
      log.md.tmpl       # log page template (first entry format)
```

No config file is generated in the repo. All defaults are embedded here.

---

## 2. SKILL.md — entry point

### Frontmatter
```yaml
name: repo-llm-wiki
description: "Generates a committable Markdown wiki from the repo it lives in — invoke with /repo-llm-wiki"
version: "0.4.0"
trigger: "/repo-llm-wiki"
compatible_agents:
  tested: [claude, cursor, copilot]
author: "Nenad Djordjevic"
github: "ndjordjevic"
license: "apache-2.0"
```

### Dispatch

1. Parse the first word after `/repo-llm-wiki` as the subcommand.
2. Route to the matching sibling file:
   - `init` → `init.md`
   - `refresh` → `refresh.md`
   - `lint` → `lint.md`
   - anything else → print usage and stop.
3. Do not proceed before reading the target file.

### Git policy (canonical, never repeated in subcommand files)

**Never run `git commit` or `git push`** unless the human explicitly asked for it in this conversation.

---

## 3. Pre-generation: repo data collection

This step runs at the start of both `init` and `refresh`, before any wiki page is written. Its purpose is to compute all facts that should never be LLM-guessed.

The agent runs the following shell commands and reads the following files. All results are held in memory for use during page generation.

### 3.1 Identity

First, verify we're inside a git working tree. If not, stop with: *"repo-llm-wiki requires a git repository. Run `git init` here first."*

```bash
git rev-parse --is-inside-work-tree         # must print "true"
git rev-parse HEAD                          # → REPO_SHA
git rev-parse --abbrev-ref HEAD             # → REPO_BRANCH
basename $(git rev-parse --show-toplevel)   # → REPO_NAME
```

### 3.2 Root-level inventory

```bash
find . -maxdepth 1 -not -name '.' -not -path '*/.git' | sort
```

Produces `ROOT_ENTRIES`. For each entry, classify as one of: `code` / `infra` / `tests` / `docs` / `scripts` / `data` / `config` / `build-artifact` / `archive` / `unclear`. Classification heuristics are in §5 (repo-map page).

### 3.3 Directory tree (3 levels)

```bash
find . -type d -maxdepth 3 \
  -not -path '*/.git/*' \
  -not -path '*/node_modules/*' \
  -not -path '*/vendor/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*' \
  | sort
```

Produces `DIR_TREE`.

### 3.4 Language and runtime versions (never trust README prose)

Search common locations and take the **first found** for each variable. If none of the candidate paths exist, leave the variable `unknown`.

| Variable | Candidate paths (first match wins) | What to extract |
|---|---|---|
| `GO_VERSION`, `MODULE_NAME` | `go.mod`, `app/go.mod`, `service/go.mod` | `^go (\S+)` and `^module (\S+)` |
| `NODE_VERSION`, `NPM_NAME` | `package.json`, `app/package.json` | `.engines.node`, `.name` |
| `TF_VERSION` | `infra/terraform.tf`, `terraform.tf`, `infra/versions.tf`, `versions.tf` | `required_version = "..."` |
| `AWS_PROVIDER_VERSION` | same as TF_VERSION | first `aws` block under `required_providers` |

### 3.5 Entry points

```bash
# Go monorepo Lambda entrypoints (look in common locations)
find ./app/cmd ./cmd -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort   # → LAMBDA_DIRS
# LAMBDA_COUNT = wc -l on the above
```

Only set `LAMBDA_DIRS` if at least one match found; otherwise leave empty.

If `LAMBDA_DIRS` is non-empty, also read each Lambda's entry file to enable the §5.3 grouping algorithm. For each `<dir>` in `LAMBDA_DIRS`:
- Read `<dir>/main.go` (or, if absent, the first `*.go` file in the dir), capped at 200 lines.
- Extract its import block lines that match `internal/(handler|service|repository)/[^"]*`. Store as `LAMBDA_IMPORTS[<dir>] = [list]`.

This is intentionally bounded: 42 Lambdas × 200 lines = ~8400 lines, comparable to one large source file.

### 3.6 Infrastructure inventory

```bash
# Environments
find infra/environments -type f -name '*.tfvars' 2>/dev/null | sort  # → ENV_FILES

# AWS resource types declared (one-liner per .tf file)
grep -r "^resource \"aws_" infra/*.tf 2>/dev/null | grep -o '"aws_[^"]*"' | sort -u
# → AWS_RESOURCE_TYPES (deduplicated list)

# GSI names — match conventional naming directly. Adjust pattern if the repo uses a different prefix.
grep -hoE '"GSI_[A-Za-z0-9_]+"' infra/*.tf 2>/dev/null | sort -u
# → GSI_NAMES (list)
# Fallback if the above returns 0: grep all `name\s*=\s*"[^"]+"` lines inside any
# `global_secondary_indexes` block; this is approximate but better than nothing.
```

### 3.7 Workflows and CI

```bash
ls .github/workflows/ 2>/dev/null | grep -E '\.ya?ml$' | sort   # → WORKFLOW_FILES
```

The `.yml`/`.yaml` filter excludes README and other non-workflow files that may live alongside.

### 3.8 Test suites

```bash
find tests -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort   # → TEST_SUITES
```

### 3.9 Key source files to read (content, not just list)

Read these files in full (up to 500 lines; for larger files read first 200 lines + any section headers):

- `README.md`
- `app/internal/model/domain.go` (if exists) — domain types
- `app/internal/model/model.gen.go` (if exists — note as generated, extract type names only)
- `infra/dynamodb.tf` (if exists)
- `infra/lambda.tf` (first 200 lines, to get Lambda function names)
- `infra/swaggers/*.yaml` or `*.json` (first 100 lines — paths and tags only)
- `.github/workflows/main.yml` first; if absent, the alphabetically first `wf-*.yml` (orchestrator workflow first; reusable `wf-*` files only when no orchestrator exists)
- `build.sh` or equivalent root build script (Makefile, `scripts/build`)

**Skip entirely** (do not read, do not reference their contents):
- `archive/` and `*_old/` dirs (note they exist in repo-map only)
- `dist/`, `build/` dirs
- `*.csv`, `*.json` data files at root (flag in repo-map, do not ingest)
- `*_test.go` / `*.test.ts` files (note test coverage exists, don't read)
- `vendor/`, `node_modules/`
- Files >1000 lines that aren't a key source file listed above

---

## 4. Profile detection

Run after §3 data collection, before any page generation. Phase 1 ships with two profiles: `iac-aws` and `generic`. Other profile names listed below are detected but treated as `generic` for page generation — they exist as labels for future phases.

**First match wins.** Evaluate in order; do not combine.

```
if AWS_RESOURCE_TYPES is non-empty OR ENV_FILES is non-empty:
    profile = "iac-aws"
    if LAMBDA_DIRS non-empty AND GO_VERSION set: flavor = "go-lambda-monorepo"
    elif LAMBDA_DIRS non-empty AND NODE_VERSION set: flavor = "node-lambda-monorepo"
    else: flavor = ""
elif GO_VERSION set:
    profile = "go-service"          # Phase 1: rendered as generic
elif NODE_VERSION set AND package.json mentions react|vue|svelte|next|nuxt:
    profile = "frontend-app"        # Phase 1: rendered as generic
elif NODE_VERSION set:
    profile = "node-service"        # Phase 1: rendered as generic
else:
    profile = "generic"
```

User override: `/repo-llm-wiki init iac-aws` forces `profile=iac-aws`.

Print the result before generation begins:
```
Detected profile: iac-aws (go-lambda-monorepo)
```

---

## 5. Page generation sequence

> **SUPERSEDED by v0.2.** The table and subsections below describe the v0.1 page set (repo-map, glossary, modules, architecture, overview, runbook). v0.2 generates: `index.md`, `lambdas.md` + `lambdas/<name>.md`, `cli.md` + `cli/<name>.md`, `migrations.md` (+ details where warranted), `scripts.md`, `infra.md` (+ `infra/<type>.md` if split), `log.md`. See `skills/repo-llm-wiki/init.md` Step 3 (categories) and Step 5 (page contracts) for authoritative detail. The sections below are kept for reference only — do not implement against them.

### v0.2 page sequence (summary; authoritative version lives in `init.md`)

| # | File | Built from | Subagent? |
|---|---|---|---|
| 1 | Category list pages (`lambdas.md`, `cli.md`, `packages.md`, `scripts.md`) | Subagent A + C + E returns | A, C, E |
| 2 | Per-item detail pages under `lambdas/`, `cli/`, `packages/` | Subagent A + E return | A, E |
| 3 | `wiki/infra.md` (+ `wiki/infra/<type>.md` if split) | Subagent B return | B |
| 4 | `wiki/log.md` | template + REPO_SHA, date, page count | — |
| 5 | `wiki/index.md` | Subagent D overview + emitted categories + tech stack from §3 | D |
| 6 | `AGENTS.md` | `templates/AGENTS_BLOCK.md` | — |

Order matters because §5 (`index.md`) lists only pages that were actually written, and `log.md`'s page count is back-filled after `index.md`.

---

### v0.1 sequence — superseded (kept below for reference)

| # | File | Input sources | LLM? |
|---|---|---|---|
| 1 | `wiki/repo-map.md` | ROOT_ENTRIES, DIR_TREE, §3 computed data | Minimal: classify + flag |
| 2 | `wiki/glossary.md` | README, domain.go, swagger | Yes |
| 3 | `wiki/modules/<name>.md` (one per module) | Per-module source files | Yes |
| 4 | `wiki/infra.md` | dynamodb.tf, lambda.tf, ENV_FILES, AWS_RESOURCE_TYPES | Yes |
| 5 | `wiki/runbook.md` | README, build.sh, workflow files | Yes |
| 6 | `wiki/architecture.md` | All of the above pages (summaries) | Yes + Mermaid |
| 7 | `wiki/overview.md` | architecture.md, runbook.md | Yes |
| 8 | `wiki/log.md` | REPO_SHA, today's date, profile, page count | No |
| 9 | `wiki/index.md` | All wiki pages now existing | Minimal: TOC |
| 10 | `AGENTS.md` | templates/AGENTS_BLOCK.md | No |

### 5.1 `wiki/repo-map.md`

Purpose: honest, warts-and-all map of the repo. Built almost entirely from computed data.

Structure:
```
# Repo Map

> Generated from directory enumeration. Does not represent intended architecture — it shows what's actually here.

## Directory tree
<DIR_TREE rendered as indented list, 3 levels>

## Root-level classification
<table: entry | type | notes>
```

Classification rules for root entries:

| Entry pattern | Type | Notes |
|---|---|---|
| `app/`, `src/`, `lib/`, `cmd/` | `code` | |
| `infra/`, `terraform/`, `tf/` | `infra` | |
| `tests/`, `test/`, `spec/` | `tests` | |
| `docs/`, `wiki/` | `docs` | |
| `*.sh`, `build-*.sh`, `Makefile` | `scripts` | list each by name |
| `*.csv`, `*.json`, `*.txt` at root | `data` | ⚠ flag (see below) |
| `archive/`, `*_old/` | `archive` | ⚠ flag as deprecated |
| `dist/`, `build/` | `build-artifact` | note: gitignored? |
| `.github/`, `.claude/`, `.cursor/` | `config` | |
| everything else | `unclear` | |

Callout blocks (emit when conditions are met):

```markdown
> **⚠ Loose files at root** — files at the repo root that don't belong to a standard project directory: `IdPermFinal 1.java`, `consents-12.json`. Consider moving or deleting.
```

```markdown
> **⚠ Likely deprecated** — `archive/`, `app/cmd/basiqToPireanMigration_old/`, `app/cmd/pfbToAMSMigration_old/`. These appear unused; confirm before referencing.
```

```markdown
> **⚠ Possibly sensitive** — files whose names suggest customer or personally identifiable data: `pccu_customer_account_*.csv`, `consents-12.json`. Verify these should be committed to the repo.
```

The "possibly sensitive" callout fires on filenames matching `*customer*`, `*consent*`, `*pii*`, `*personal*`, `*account*` with extensions `.csv`, `.json`, `.txt`.

### 5.2 `wiki/glossary.md`

Read: README, domain model file, API spec (paths + tags only).

**Term selection heuristic:** include a term only if it satisfies all three:
1. Appears with a capital letter (proper noun) or is a multi-word concept.
2. Appears at least 3 times across the inputs (README + domain model + API spec).
3. Is either (a) a type/struct name in the domain model, (b) a noun in the API path/tag, or (c) explicitly defined in README prose.

Cap at ~25 terms — pick the most-cited if the heuristic produces more. If <3 terms qualify, write a placeholder line `> No domain glossary identified at generation time.` and skip the page (do not write an empty file).

For each kept term: write a definition with one citation. Keep definitions to 1–3 sentences. Do not invent terms not found in the source files.

Format per term:
```markdown
### Authorisation
A record of a customer's consent for a Data Holder to share specified data with a Data Recipient.
([app/internal/model/domain.go](../app/internal/model/domain.go))
```

### 5.3 `wiki/modules/<name>.md`

One page per distinct top-level module. For repos with many entrypoints (e.g. `go-lambda-monorepo` with 40+ `cmd/` dirs), group them — one page per Lambda is too granular, one page for everything is useless.

#### Grouping algorithm (deterministic, applied in order)

1. **By shared internal package.** Use `LAMBDA_IMPORTS` collected in §3.5. Cluster Lambdas that share their primary `internal/service/<X>` (or `internal/handler/<X>`) import. Group name = `<X>`.
2. **By name prefix.** For Lambdas not yet grouped, group by longest common prefix ≥3 chars (`migrationProcessing`, `migrationInitiation`, `migrationValidation` → `migration`).
3. **`_old`/`_v1` suffix items.** If a deprecated-suffix Lambda has a peer with the matching root name, group with that peer and mark the group as containing deprecated members. If no peer exists (e.g. `basiqToPireanMigration_old` with no `basiqToPireanMigration`), group with the closest prefix match from rule 2; if no prefix match either, treat as a singleton.
4. **Singletons** become their own group.

Target: **3–8 module pages** for any monorepo. If the algorithm yields >8 groups, merge the two smallest by name similarity until ≤8. If <3, keep as-is.

Page slug: kebab-case of the group name (`authorisation-lifecycle`, `migration-pipeline`).

#### What each module page contains

- **What it does** — 1 paragraph, cited.
- **Members** — table of Lambdas / files in this group, each linked to its source dir. Include a `deprecated` column if any members were grouped via rule 3.
- **Key files** — handler / service / repository files this group reads from `app/internal/`, with citations.
- **External dependencies** — DynamoDB tables/GSIs (from `GSI_NAMES` filtered by which appear in source), SQS queues, external API calls. Each dependency cites a Terraform resource or SDK import line.
- **Representative entrypoints read** — explicit list of which Lambda main files were sampled (per §6 batching).
- **Confidence note** if anything is inferred.

For `generic` and other non-IaC profiles, "Lambdas" becomes "entrypoints" or "subcommands" depending on what was found.

### 5.4 `wiki/infra.md` (iac-aws profile only)

For non-`iac-aws` profiles in Phase 1, **skip this page entirely** — do not create a stub. Other infra forms (Docker, Helm, K8s, Pulumi) are out of scope for Phase 1.

Structure:
```
## AWS Resources
<table derived from AWS_RESOURCE_TYPES: resource type | count | key names>

## DynamoDB
<table: table name | primary key | GSI list from GSI_NAMES>

## Environments
<list from ENV_FILES>

## Deployment modes
<extracted from README deployment section or wf-deploy.yml>

## IAM and security
<any iam_role or security group resources found in *.tf>
```

All counts and names come from §3 computed data. LLM's job: structure and explain, not enumerate.

### 5.5 `wiki/runbook.md`

Extract from README (Development Setup, Testing, Deployment sections) and `build.sh`.

Structure:
```
## Prerequisites
## Local setup
## Run tests
## Build
## Deploy
## Useful scripts
<table: script | what it does (1 line)>
```

The "useful scripts" table lists the `build-*.sh` files at the repo root with one-line descriptions inferred from their names and a brief read of their content.

### 5.6 `wiki/architecture.md`

Synthesizes from already-written module and infra pages. Does not re-read source files.

**Skip condition:** if the repo has no `infra.md` AND fewer than 2 module pages (i.e. nothing to draw a diagram of), skip this page. Index will note the absence.

Structure:
```
## System overview
<1–2 paragraphs>

## Component diagram
```mermaid
flowchart TD
    ...
```
<!-- draft: verify arrows before sharing externally -->

## Components
<table: component | role | wiki page>

## External dependencies
<table: service | purpose | evidence file>

## Data flow
<narrative of the main happy path, citing module pages>
```

Mermaid rules:
- Maximum 10 nodes (keep readable)
- Only include components with evidence in the repo
- Every arrow must correspond to a verified dependency (Terraform resource, SDK import, or explicit README claim)
- Always add the `<!-- draft -->` comment

### 5.7 `wiki/overview.md`

200–400 words. Plain language. No jargon without a `[[glossary]]` link.

Structure:
```
## What is this?
<1 paragraph>

## Who uses it?
<1 paragraph>

## Key capabilities
<bulleted list, 4–6 items>

## Tech stack
<table: component | technology — derived from computed version variables>

## Where to go next
<links to architecture.md, runbook.md, index.md>
```

Version data comes exclusively from §3 computed variables (`GO_VERSION`, `TF_VERSION`, etc.) — never from README prose.

### 5.8 `wiki/log.md`

Append-only. First entry format:
```
## 2026-05-02 14:23 UTC — init
- SHA: abc1234
- Branch: main
- Profile: iac-aws (go-lambda-monorepo)
- Pages generated: 12
- Tool: repo-llm-wiki v0.1.0
```

On `refresh`, append a new entry (never overwrite).

### 5.9 `wiki/index.md`

Last page written. Generated from the list of pages now in `wiki/`.

To compute the "N commits ahead" figure (omit if zero):
```bash
git rev-list --count <log-sha>..HEAD     # → COMMITS_AHEAD
```
Where `<log-sha>` is the SHA from the most recent `## ` entry in `wiki/log.md`. If the SHA cannot be parsed or the rev-list call fails, omit the staleness line entirely.

Structure:
```
# <REPO_NAME> wiki

> Last refreshed: 2026-05-02 — SHA abc1234 — ⚠ HEAD is N commits ahead, consider running `/repo-llm-wiki refresh`
> (omit the warning if HEAD == log SHA)

## Start here
- [[overview]] — what this repo is
- [[architecture]] — components and data flow
- [[runbook]] — build, test, deploy

## Module reference
- [[modules/authorisation-lifecycle]]
- ...

## Infrastructure
- [[infra]]

## Reference
- [[glossary]]
- [[repo-map]]

## Meta
- [[log]] — generation history
```

All links are `[[wikilinks]]`. No external links in index.md.

### 5.10 `AGENTS.md` — append block

If `AGENTS.md` does not exist: create it with only this block.
If it exists: append the block, separated by a blank line before and after.

Block format (from `templates/AGENTS_BLOCK.md`):
```markdown
<!-- repo-llm-wiki: begin -->
## Repo wiki

This repo has a generated wiki in `wiki/`. Before answering questions about
architecture, debugging, or features, read `wiki/index.md` and follow
[[wikilinks]] into relevant pages.

Rules:
- Cite wiki page names in answers (e.g. "per [[modules/authorisation-lifecycle]]").
- If the wiki does not contain an answer, say so and then look at the source code directly.
- Do not run `git commit` or `git push` unless explicitly asked.
<!-- repo-llm-wiki: end -->
```

On `refresh`, check whether the block already exists (look for `<!-- repo-llm-wiki: begin -->`). If found, leave it unchanged.

---

## 6. Token budgeting

These rules are embedded in the `init.md` and `refresh.md` instructions so the agent follows them automatically.

### Read limits per file
- Files ≤300 lines: read in full.
- Files 300–1000 lines: read in full, but flag that it's large.
- Files >1000 lines: read first 300 lines. If it's a key file (domain model, main TF file), also grep for key patterns (`^func `, `^type `, `^resource `, `global_secondary_index`) and include those matches.

### What not to read at all
`archive/`, `*_old*/`, `dist/`, `build/`, `vendor/`, `node_modules/`, `*.sum`, `*.lock` (lock files), `*.tfstate`, `*_test.go`, `*.test.ts`, `*.csv`, `*.json` at root, binary files.

### Batching module reads
Do not read all 42 Lambdas. Read the entrypoint `main.go` (or equivalent) for 2–3 representative Lambdas per group, then write the group page. Call this out in the page: *"This page describes the authorisation lifecycle group (5 Lambdas). Representative entrypoints read: `createAuthorisation`, `revokeAuthorisation`."*

---

## 7. Accuracy enforcement (embedded in instructions)

Rules the skill instructions impose on the agent at generation time:

1. **Versions from source**: always use `GO_VERSION` / `TF_VERSION` / `NODE_VERSION` from §3. If none found, write `unknown` — never trust README prose.
2. **Counts from shell**: Lambda count = `LAMBDA_COUNT`. GSI count = length of `GSI_NAMES`. Never estimate.
3. **No invented names**: every component name in `architecture.md` must match a real directory, file, or Terraform resource. If unsure, omit and add a confidence note.
4. **Source-file citations forbidden (v0.2+ inversion)**: relative paths to source files (`../app/...`) must NOT appear in any wiki page. Obsidian opens `wiki/` as the vault root so they resolve to nothing. Lint E2 errors on any `../` or `./` link leaving `wiki/`. The v0.1 rule that required citations was reversed.
5. **Confidence markers**: when a claim is inferred (no explicit source), add `> Confidence: low — inferred from directory names.` Do not silently state low-confidence things as fact.
6. **Sensitive flags without reading content**: the "possibly sensitive" callout in `repo-map.md` fires on filename patterns **and** an extension restricted to `.csv`, `.json`, `.txt`, `.xlsx`, `.xls`, `.sql`, `.dump`. Source code files matching the same name patterns (e.g. `account_handler.go`) are never flagged. The agent does not read the contents of any flagged file.

---

## 8. `init.md` — detailed steps

```
Guard A: if wiki/index.md already exists → stop.
  Tell user: "A wiki already exists. Run /repo-llm-wiki refresh to regenerate."

Guard B: if wiki/ exists with content but no index.md → stop.
  Tell user: "A wiki/ directory exists with N files but no index.md.
              Move or remove it before running /repo-llm-wiki init."
  (Forcing past this guard is out of scope for Phase 1.)

Step 1: Run repo data collection (§3). Print: "Collecting repo data..."
Step 2: Detect profile (§4). Print detected profile.
Step 3: Create wiki/ and wiki/modules/ directories.
Step 4: Generate pages in order (§5.1–5.10), printing the page name as each is written.
        If a page is skipped per its skip condition, print "skipped: <page> (<reason>)".
Step 5: Print completion summary (see below).
```

**Atomicity:** Phase 1 writes pages directly. If a step fails mid-generation, the wiki is in a partial state. Acceptable risk: the agent's stop point will be visible from which pages exist; re-running `init` will fail Guard A, telling the user to remove the partial output and try again. A future revision may write to `.wiki.tmp/` then atomic-move.

Completion summary format (list only pages actually written; omit skipped ones):
```
Wiki generated in wiki/ (N pages)

  wiki/index.md          ← start here
  wiki/overview.md
  wiki/architecture.md
  wiki/modules/          (M pages)
  wiki/infra.md          (iac-aws profile only)
  wiki/runbook.md
  wiki/repo-map.md
  wiki/glossary.md
  wiki/log.md
  AGENTS.md              ← extended with wiki block

Profile: iac-aws (go-lambda-monorepo)
SHA: abc1234  Branch: main

Next: review wiki/index.md, then commit wiki/ to git.
Run /repo-llm-wiki lint to validate.
```

`N` = total pages written (count modules individually). `M` = module page count.

---

## 9. `refresh.md` — detailed steps

```
Guard: if wiki/index.md does NOT exist → stop.
Tell user: "No wiki found. Run /repo-llm-wiki init first."

Step 1: Run repo data collection (§3). Print: "Collecting repo data..."
Step 2: Detect profile (§4).
Step 3: Regenerate all pages in order (§5.1–5.10).
         Overwrite all pages except log.md (which gets a new entry appended).
         Do not touch AGENTS.md wiki block if it already exists.
Step 4: Print completion summary (same format as init, with "refreshed" wording).
```

---

## 10. `lint.md` — checks

Run all checks. Collect all warnings and errors. Print a grouped report at the end.

### Errors (must fix, wiki is not valid)

| Check | How |
|---|---|
| Broken wikilinks | For every `[[slug]]` in wiki/, check `wiki/<slug>.md` exists |
| Forbidden file citations (E2) | For every `(../path)` or `(./path)` in wiki/, **error** — relative paths leaving `wiki/` are forbidden (Obsidian vault root issue). Was "check they resolve" in v0.1; inverted in v0.2. |
| Missing log entry | `wiki/log.md` must have at least one `## ` entry |
| Missing AGENTS.md block | `AGENTS.md` must contain `<!-- repo-llm-wiki: begin -->` |
| index.md missing links | Every `.md` file in `wiki/` (except `log.md`, `.archive/`) must appear in `wiki/index.md` |

### Warnings (flag but don't fail)

| Check | How |
|---|---|
| Stale wiki | Parse SHA from first `## ` entry of `wiki/log.md`; run `git rev-list --count <sha>..HEAD`. Warn if result >20. |
| Uncited pages | For each `wiki/modules/*.md` and `wiki/architecture.md`, count `([` citation occurrences. Warn if zero. (This is a deterministic per-page rule. Per-paragraph quality requires human judgment and is out of scope for lint.) |
| Draft mermaid | If `architecture.md` contains `<!-- draft -->`, remind the author the diagram needs review before sharing externally. |
| Confidence: low markers | List pages containing `Confidence: low` so the author knows what to verify. |

### Mermaid syntax check (regex-only, not a real parser)

The skill cannot parse Mermaid. For each ` ```mermaid ` block: check via regex that the first non-blank line matches one of the known diagram-type keywords (`flowchart`, `graph`, `sequenceDiagram`, `classDiagram`, `stateDiagram`, `erDiagram`, `gantt`, `pie`) and that the block contains at least one line with `-->` or `---` or `:`. Report blocks failing both checks as errors. This catches obvious corruption but does not validate semantics.

---

## 11. Test plan

### Validation repo

Run against `tfaws-dregdata-dhs-app-authorisation`. Acceptance criteria from PRD §7.

Specific facts to verify post-generation:
- `wiki/overview.md` tech stack table shows Go **1.24**, not 1.23
- `wiki/infra.md` DynamoDB section lists all 7 GSI names by name
- `wiki/modules/` has between 3 and 8 grouped pages (not 42 individual ones)
- One of those module pages covers the migration pipeline (regardless of exact name)
- `wiki/repo-map.md` includes the 3 callout blocks (loose files, deprecated dirs, possibly sensitive)
- `wiki/runbook.md` lists all 8 workflow `.yml` files (README.MD is not a workflow)
- `wiki/log.md` has a valid first entry whose SHA matches `git rev-parse HEAD` at generation time
- `/repo-llm-wiki lint` produces 0 errors
- `AGENTS.md` contains the `<!-- repo-llm-wiki: begin -->` … `<!-- repo-llm-wiki: end -->` block

### Regression repo

Pick one additional repo of `generic` profile. Confirm:
- Init completes without error
- No `infra.md` generated (no Terraform found)
- Lint passes

### Determinism check

Run `refresh` twice on an unchanged repo. Diff:
- `wiki/repo-map.md` — must be identical
- `wiki/log.md` — must gain exactly one new entry, no other changes
- `wiki/index.md` TOC links — must be identical
- `wiki/overview.md` — acceptable to have prose variation; flag if >20% of lines differ (matches PRD §7 criterion 6)
