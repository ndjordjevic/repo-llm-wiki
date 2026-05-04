# init — scaffold and generate a new wiki

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

If a profile arg was passed (`/repo-llm-wiki init iac-aws`), set `PROFILE_OVERRIDE = iac-aws`. Valid values: `iac-aws`, `generic`. If invalid, stop with the SKILL.md usage block.

---

## Guards (run all three before any work)

**Guard A** — if `wiki/index.md` already exists in the current working directory, **stop**:
> "A wiki already exists here (`wiki/index.md` found). Run `/repo-llm-wiki refresh` to regenerate."

**Guard B** — if `wiki/` exists with content but no `wiki/index.md`, **stop**:
> "A `wiki/` directory exists with N file(s) but no `index.md`. Move or remove it before running `/repo-llm-wiki init`."

**Guard C** — verify we're inside a git working tree:
```bash
git rev-parse --is-inside-work-tree
```
If this returns anything other than `true`, **stop**:
> "repo-llm-wiki requires a git repository. Run `git init` here first."

---

## Step 1 — Collect repo data

Print: *"Collecting repo data..."*

Run all of the following. Hold results in memory under the variable names shown. **Skip a sub-step if its inputs don't exist; never error.** When a variable has no source, leave it `unknown` (versions) or empty (lists).

### 1.1 Identity

```bash
git rev-parse HEAD                          # → REPO_SHA
git rev-parse --abbrev-ref HEAD             # → REPO_BRANCH
basename "$(git rev-parse --show-toplevel)" # → REPO_NAME
```

### 1.2 Root-level inventory

```bash
find . -maxdepth 1 -not -name '.' -not -path '*/.git' | sort   # → ROOT_ENTRIES
```

For each entry, classify per the table in §2.6 of this file (repo-map page).

### 1.3 Directory tree (3 levels)

```bash
find . -type d -maxdepth 3 \
  -not -path '*/.git/*' \
  -not -path '*/node_modules/*' \
  -not -path '*/vendor/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*' \
  | sort
# → DIR_TREE
```

### 1.4 Language and runtime versions

Search candidate paths in order; **first match wins**. If none exist, leave `unknown`.

| Variable | Candidate paths | What to extract |
|---|---|---|
| `GO_VERSION`, `MODULE_NAME` | `go.mod`, `app/go.mod`, `service/go.mod` | `^go (\S+)` and `^module (\S+)` |
| `NODE_VERSION`, `NPM_NAME` | `package.json`, `app/package.json` | `.engines.node`, `.name` |
| `TF_VERSION` | `infra/terraform.tf`, `terraform.tf`, `infra/versions.tf`, `versions.tf` | `required_version = "..."` |
| `AWS_PROVIDER_VERSION` | same files as TF_VERSION | first `aws { ... version = "..." }` block under `required_providers` |

**Never** read these from README prose. Always from the source manifest.

### 1.5 Entry points

```bash
find ./app/cmd ./cmd -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort   # → CMD_DIRS
```

For each `<dir>` in `CMD_DIRS`:
- Read `<dir>/main.go` (or, if absent, the first `*.go` in the dir), capped at 200 lines.
- If the file contains `aws-lambda-go` import or `lambda.Start` call → classify as **Lambda**, add to `LAMBDA_DIRS`.
- Otherwise → classify as **CLI tool**, add to `CLI_DIRS`.
- Extract import lines matching `internal/(handler|service|repository)/[^"]*` and store as `CMD_IMPORTS[<dir>]` (used for module grouping regardless of Lambda vs CLI).

`LAMBDA_COUNT` = count of `LAMBDA_DIRS`.  
`CLI_COUNT` = count of `CLI_DIRS`.

**Do not use `CMD_DIRS` count as `LAMBDA_COUNT`** — not every binary under `cmd/` is an AWS Lambda function. CLI tools, migration runners, and test helpers live alongside Lambdas in `cmd/` and must be labelled correctly.

This is bounded: ~50 dirs × 200 lines = ~10k lines, comparable to one large source file.

### 1.6 Infrastructure inventory

```bash
# Environments
find infra/environments -type f -name '*.tfvars' 2>/dev/null | sort
# → ENV_FILES

# AWS resource types
grep -rh "^resource \"aws_" infra/*.tf 2>/dev/null | grep -oE '"aws_[^"]+"' | sort -u
# → AWS_RESOURCE_TYPES

# GSI names — match conventional naming directly
grep -hoE '"GSI_[A-Za-z0-9_]+"' infra/*.tf 2>/dev/null | sort -u
# → GSI_NAMES
# Fallback if zero results: read dynamodb.tf and grep `name\s*=\s*"[^"]+"` lines that
# appear inside `global_secondary_indexes` / `global_secondary_index` blocks.
```

### 1.7 Workflows and CI

```bash
ls .github/workflows/ 2>/dev/null | grep -E '\.ya?ml$' | sort   # → WORKFLOW_FILES
```

The `.yml`/`.yaml` filter excludes README and other non-workflow files that may live alongside.

### 1.8 Test suites

```bash
find tests -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort   # → TEST_SUITES
```

### 1.9 Read key source files

Read in full (≤500 lines; for larger files read first 200 lines):

- `README.md`
- `app/internal/model/domain.go` (if exists) — domain types
- `app/internal/model/model.gen.go` (if exists) — note as generated; extract type names only
- `infra/dynamodb.tf` (if exists)
- `infra/lambda.tf` (first 200 lines, for Lambda function names)
- `infra/swaggers/*.yaml` or `*.json` (first 100 lines — paths and tags only)
- `.github/workflows/main.yml` first; if absent, the alphabetically first `wf-*.yml`
- `build.sh` or equivalent root build script (Makefile, `scripts/build`)

**Skip entirely** (do not read):
- `archive/` and `*_old*/` dirs (note they exist in repo-map only)
- `dist/`, `build/` dirs
- `*.csv`, `*.json` data files at root
- `*_test.go`, `*.test.ts`
- `vendor/`, `node_modules/`
- `*.sum`, `*.lock`, `*.tfstate`, binaries
- Files >1000 lines that aren't a key source file listed above

---

## Step 2 — Detect profile

Print: *"Detecting profile..."*

If `PROFILE_OVERRIDE` was set, use it. Otherwise, evaluate in order; **first match wins**:

```
if AWS_RESOURCE_TYPES non-empty OR ENV_FILES non-empty:
    PROFILE = "iac-aws"
    if LAMBDA_DIRS non-empty AND GO_VERSION set:    FLAVOR = "go-lambda-monorepo"
    elif LAMBDA_DIRS non-empty AND NODE_VERSION set: FLAVOR = "node-lambda-monorepo"
    else: FLAVOR = ""
elif GO_VERSION set:
    PROFILE = "generic"     # labelled go-service internally
elif NODE_VERSION set AND package.json mentions react|vue|svelte|next|nuxt:
    PROFILE = "generic"     # labelled frontend-app internally
elif NODE_VERSION set:
    PROFILE = "generic"     # labelled node-service internally
else:
    PROFILE = "generic"
```

Phase 1 only renders two profiles distinctly: `iac-aws` and `generic`.

Print:
```
Detected profile: <PROFILE> [<FLAVOR>]
```

---

## Step 3 — Create directories

```bash
mkdir -p wiki wiki/modules wiki/.archive
```

---

## Step 4 — Generate pages in order

Pages are written in this exact order. Later pages reference earlier ones; never reorder.

For each page below, after writing it print: `wrote: wiki/<path>`. If a page is skipped, print: `skipped: wiki/<path> (<reason>)`. Track `PAGES_WRITTEN_COUNT`.

### 4.1 `wiki/repo-map.md`

Purpose: honest, warts-and-all map of the repo. Built almost entirely from computed data.

Render the directory tree from `DIR_TREE` as an indented Markdown list (3 levels max, sorted, one entry per line, indent by depth).

**Root-level classification table** — for each entry in `ROOT_ENTRIES`, produce one row:

| Entry pattern | Type | Notes column |
|---|---|---|
| `app/`, `src/`, `lib/`, `cmd/` | `code` | (empty) |
| `infra/`, `terraform/`, `tf/` | `infra` | (empty) |
| `tests/`, `test/`, `spec/` | `tests` | (empty) |
| `docs/`, `wiki/` | `docs` | (empty) |
| `*.sh`, `build-*.sh`, `Makefile` | `scripts` | filename |
| `*.csv`, `*.json`, `*.txt`, `*.xlsx`, `*.xls`, `*.sql`, `*.dump` at root | `data` | "⚠ flagged" if name matches sensitive patterns (see callout 3) |
| `archive/`, `*_old/` | `archive` | "⚠ deprecated" |
| `dist/`, `build/`, `out/`, `target/` | `build-artifact` | (empty) |
| `.github/`, `.claude/`, `.cursor/`, `.devcontainer/`, `.idea/`, `.vscode/` | `config` | (empty) |
| anything else | `unclear` | (empty) |

**Callout blocks** (emit each only when its condition matches):

1. **⚠ Loose files at root** — root-level files that aren't in a standard project directory and aren't standard project metadata (`README*`, `LICENSE*`, `CODEOWNERS`, `.gitignore`, `Makefile`, `*.sh`, `package.json`, `go.mod`, `Dockerfile`, `*.tf`). List each by name. If none, omit the callout.

2. **⚠ Likely deprecated** — any path matching `archive/`, `**/*_old/`, `**/*_old*`, `**/*_v1/`, `**/*_legacy/`. List each. If none, omit.

3. **⚠ Possibly sensitive** — files whose **filename** matches `*customer*`, `*consent*`, `*pii*`, `*personal*`, `*account*`, `*ppid*`, `*identifier*` AND whose **extension** is one of `.csv`, `.json`, `.txt`, `.xlsx`, `.xls`, `.sql`, `.dump`. **Do not read the contents** of these files. Source code files (`.go`, `.py`, `.ts`, …) are never flagged here even if their names match. List each by relative path. If none, omit.

Output structure:
```markdown
# Repo Map

> Generated from directory enumeration. Does not represent intended architecture — it shows what's actually here.

## Directory tree
<rendered DIR_TREE>

## Root-level classification
<table>

<callouts, in order, only those that fired>
```

### 4.2 `wiki/glossary.md`

Read: README, `app/internal/model/domain.go` (or analogous domain file), API spec paths/tags only.

**Term selection** — include a term only if:
1. It's capitalised (proper noun) or a multi-word concept, AND
2. It appears at least 3 times across the inputs combined, AND
3. It is one of: a type/struct name in the domain model, a noun in an API path/tag, or explicitly defined in README prose.

Cap at 25 terms (most-cited wins). If fewer than 3 qualify, **skip the page** entirely (do not write a stub) and print: `skipped: wiki/glossary.md (no domain terms identified)`.

For each kept term, write 1–3 sentences with one citation. Format:
```markdown
### Authorisation
A record of a customer's consent for a Data Holder to share specified data with a Data Recipient.
([app/internal/model/domain.go](../app/internal/model/domain.go))
```

### 4.3 `wiki/modules/<slug>.md`

#### Grouping algorithm (deterministic; apply in order)

Goal: produce 3–8 module pages, never one per Lambda.

1. **By shared internal package.** Cluster `CMD_DIRS` whose `CMD_IMPORTS` share their primary `internal/service/<X>` (or `internal/handler/<X>`) import. Group name = `<X>`.
2. **By name prefix.** For entries not yet grouped, group by longest common prefix ≥3 chars (e.g. `migrationProcessing`, `migrationInitiation`, `migrationValidation` → `migration`).
3. **`_old`/`_v1` suffix items.** If a deprecated-suffix entry has a peer with the matching root name, group with that peer and mark the group as containing deprecated members. If no peer exists, group with the closest prefix match from rule 2; if no prefix match either, treat as a singleton.
4. **Singletons** become their own group.

If after these rules there are >8 groups, merge the two smallest by name similarity until ≤8. If <3 and `CMD_DIRS` is non-empty, keep as-is.

For non-monorepo profiles, "module" means top-level package or service directory in `src/`/`lib/`. For `generic` with no clear modules, write a single `wiki/modules/main.md`.

Page slug: kebab-case of the group name (`authorisation-lifecycle`, `migration-pipeline`).

#### What each module page contains

```markdown
# <Group Name>

> Confidence: <high|medium|low> — <why>

<1 paragraph: what this group does, with at least one citation>

## Members
| Entry | Type | Source | Deprecated |
|---|---|---|---|
| createAuthorisation | Lambda | [app/cmd/createAuthorisation/](../../app/cmd/createAuthorisation/) | |
| migrationVerification | CLI tool | [app/cmd/migrationverification/](../../app/cmd/migrationverification/) | |
| ... | ... | ... | ✓ (if applicable) |

## Key files
- [app/internal/handler/authorisation/handler.go](../../app/internal/handler/authorisation/handler.go) — purpose
- ...

## External dependencies
| Service | Purpose | Evidence |
|---|---|---|
| DynamoDB GSI_CustomerAuthorisations | reads by customer | [infra/dynamodb.tf](../../infra/dynamodb.tf) |
| SQS authorisation-events | publishes lifecycle events | [infra/sqs.tf](../../infra/sqs.tf) |

## Representative entrypoints read
- `<lambda1>/main.go`
- `<lambda2>/main.go`
```

### 4.4 `wiki/infra.md`

**Skip this page if `PROFILE != "iac-aws"`.** Print: `skipped: wiki/infra.md (profile is generic)` and continue.

For `iac-aws`:
```markdown
# Infrastructure

## AWS Resources
<table from AWS_RESOURCE_TYPES: type | count | example name(s) — at most 3>

## DynamoDB
<table: table | partition key | GSIs (from GSI_NAMES, all of them, comma-separated)>

## Environments
<bulleted list from ENV_FILES, one per line>

## Deployment modes
<extracted from README "Deployment" section if present, else from the orchestrator workflow file>

## IAM and security
<list any aws_iam_role / aws_security_group resources, citing the .tf file>
```

All counts and names come from §1 computed data. The agent's job is structure and prose — **not enumeration**.

### 4.5 `wiki/runbook.md`

Extract from README (Setup / Testing / Deployment sections) and `build.sh` (or Makefile).

**Version sourcing rule**: tool/runtime versions in `## Prerequisites` come **exclusively** from §1 computed variables (`GO_VERSION`, `NODE_VERSION`, `TF_VERSION`, `AWS_PROVIDER_VERSION`). Never copy version numbers from README prose — README text drifts from `go.mod`/`terraform.tf` and that staleness is the #1 failure mode this skill exists to beat. Phrase as exact version (e.g. "Go 1.24.0") not "or later".

```markdown
# Runbook

## Prerequisites
<list tools with versions from §1 variables; cite the manifest file (e.g. ../app/go.mod), not the README>

## Local setup
<command block from README>

## Run tests
<command block>

## Build
<command block; cite build.sh or Makefile>

## Deploy
<command block; cite deployment workflow>

## Workflows
<table from WORKFLOW_FILES: file | purpose (one line, inferred from name + first 20 lines if read)>

## Useful scripts
<table of root-level *.sh files: script | one-line purpose inferred from filename>
```

### 4.6 `wiki/architecture.md`

**Skip condition**: if no `infra.md` was written AND fewer than 2 module pages exist, skip with: `skipped: wiki/architecture.md (insufficient material)`.

Synthesizes from already-written module and infra pages. **Do not re-read source files.**

```markdown
# Architecture

## System overview
<1–2 paragraphs>

## Component diagram
```mermaid
flowchart TD
    <≤10 nodes; only components with evidence in this repo>
    <every arrow corresponds to a verified dependency>
```
<!-- draft: verify arrows before sharing externally -->

## Components
<table: component | role | wiki page link>

## External dependencies
<table: service | purpose | evidence file>

## Data flow
<narrative of the main happy path, citing module pages with [[wikilinks]]>
```

Mermaid rules: max 10 nodes; only components with evidence in this repo; every arrow corresponds to a Terraform resource, SDK import, or explicit README claim. Always include the `<!-- draft -->` HTML comment.

### 4.7 `wiki/overview.md`

200–400 words. Plain language. No jargon without a `[[glossary]]` link. Version data **exclusively** from §1 computed variables — never from README prose.

```markdown
# Overview

## What is this?
<1 paragraph>

## Who uses it?
<1 paragraph>

## Key capabilities
- ...
- (4–6 bullets)

## Tech stack
| Component | Technology |
|---|---|
| Runtime | <GO_VERSION or NODE_VERSION> |
| Infrastructure | <Terraform <TF_VERSION>, AWS provider <AWS_PROVIDER_VERSION>, ...> |
| Persistence | <inferred from AWS_RESOURCE_TYPES> |
| Compute | <AWS Lambda (<N> functions) — use `aws_lambda_function` count from AWS_RESOURCE_TYPES as the authoritative Lambda count, not the total number of cmd/ directories (which also includes CLI tools and migration scripts)> |

## Where to go next
- [[architecture]]
- [[runbook]]
- [[index]]
```

### 4.8 `wiki/log.md`

Read `<skill-dir>/templates/wiki/log.md.tmpl`. Substitute:
- `{{TIMESTAMP}}` → current UTC timestamp, format `YYYY-MM-DD HH:MM UTC`
- `{{COMMAND}}` → `init`
- `{{SHA}}` → first 7 chars of `REPO_SHA`
- `{{BRANCH}}` → `REPO_BRANCH`
- `{{PROFILE}}` → `<PROFILE> [(<FLAVOR>)]` (omit flavor parens if empty)
- `{{PAGE_COUNT}}` → write the placeholder `__PAGE_COUNT__` for now; after `index.md` is written, count all `*.md` files inside `wiki/` (recursively), then `sed -i` (or equivalent) to replace `__PAGE_COUNT__` with the final number in `wiki/log.md`. This ensures log and completion summary agree.
- `{{TOOL_VERSION}}` → `0.1.0` (from SKILL.md frontmatter)

Write to `wiki/log.md`.

### 4.9 `wiki/index.md`

Last wiki page written. Compute the staleness suffix:

```bash
git rev-list --count <REPO_SHA>..HEAD     # → COMMITS_AHEAD
```

Since this is `init`, `REPO_SHA == HEAD`, so `COMMITS_AHEAD == 0` and the staleness suffix is empty. (`refresh.md` uses the same template with non-zero values when applicable.)

Read `<skill-dir>/templates/wiki/index.md.tmpl`. Substitute:
- `{{REPO_NAME}}` → `REPO_NAME`
- `{{DATE}}` → today, `YYYY-MM-DD`
- `{{SHA}}` → first 7 chars of `REPO_SHA`
- `{{STALE_SUFFIX}}` → empty (init case)
- `{{START_HERE_LINKS}}` → bulleted list, only of pages actually written:
  - `- [[overview]] — what this repo is` (only if `wiki/overview.md` exists)
  - `- [[architecture]] — components and data flow` (only if exists)
  - `- [[runbook]] — build, test, deploy` (only if exists)
- `{{MODULE_LINKS}}` → bulleted list of `- [[modules/<slug>]]` for each module page actually written
- `{{INFRA_SECTION}}` → if `wiki/infra.md` exists: `\n## Infrastructure\n- [[infra]]\n` (with leading newline so it spaces correctly under the module list); else empty string
- `{{REFERENCE_LINKS}}` → bulleted list, only of pages actually written:
  - `- [[glossary]]` (only if `wiki/glossary.md` exists)
  - `- [[repo-map]]` (always — repo-map is never skipped)

All links are `[[wikilinks]]`. **No external links** in `index.md`. Never include a wikilink to a page that wasn't written — lint E1 will fail.

### 4.10 `AGENTS.md` (or extension)

Read `<skill-dir>/templates/AGENTS_BLOCK.md`. Call its content `BLOCK`.

If `AGENTS.md` does not exist in the repo root: write `BLOCK` as the entire file contents.

If `AGENTS.md` exists:
1. Check whether it already contains `<!-- repo-llm-wiki: begin -->`. If yes, leave the file untouched and print: `wiki block already present in AGENTS.md`.
2. Otherwise, append a single blank line followed by `BLOCK` (with its own trailing newline) to the end of the file.

Never replace or rewrite an existing `AGENTS.md`.

---

## Step 5 — Print completion summary

```
Wiki generated in wiki/ (<PAGES_WRITTEN_COUNT> pages)

  wiki/index.md          ← start here
  wiki/overview.md
  wiki/architecture.md      <list only if not skipped>
  wiki/modules/          (<M> pages)
  wiki/infra.md             <list only if not skipped>
  wiki/runbook.md
  wiki/repo-map.md
  wiki/glossary.md          <list only if not skipped>
  wiki/log.md
  AGENTS.md              ← extended with wiki block

Profile: <PROFILE> [<FLAVOR>]
SHA: <REPO_SHA[0:7]>  Branch: <REPO_BRANCH>

Next: review wiki/index.md, then commit wiki/ to git.
Run /repo-llm-wiki lint to validate.
```

`<M>` = number of pages in `wiki/modules/`. Only list pages that were actually written (omit any that printed `skipped:`).

---

## Atomicity note

Phase 1 writes pages directly to `wiki/`. If a step fails mid-generation, the wiki is in a partial state. Recovery: the user removes `wiki/` and re-runs `init`. A future revision may write to `.wiki.tmp/` and atomic-move on success.
