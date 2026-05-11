# init — scaffold and generate a new wiki

(Skill-directory paths are defined in `SKILL.md`. The git policy in `SKILL.md` is canonical: do not commit or push unless the human explicitly asked.)

---

## Design intent (read first)

The wiki is **small and drill-down**. From `index.md` a reader picks a category (e.g. *Lambdas*), reaches a category page listing items, clicks an item and lands on a short page describing *what that thing does* — a 2-minute read, no source-file links, just a flow narrative.

Hard rules that shape every step below:

- **Only `[[wikilinks]]` between wiki pages.** No relative paths to source files (`../app/cmd/...`). Obsidian opens `wiki/` as the vault root and source-file paths resolve to nothing; that's why we drop them.
- **Fixed category set. Do not invent categories.** The wiki has exactly four possible top-level categories: `lambdas`, `cli`, `scripts`, `infra`. A category page is emitted **only** when its source content exists in the repo; otherwise it is skipped. Never invent additional categories (no `migrations`, no `services`, no `tools`, no `pipelines`) — migration Lambdas go in `lambdas`, migration CLI tools go in `cli`, deprecated items are tagged inline within their natural category.
- **One starting page only: `index.md`.** No separate `overview.md`, `architecture.md`, `repo-map.md`, or `glossary.md`. The overview paragraph lives in `index.md`.
- **Per-item pages describe a flow, not files.** For a Lambda: who triggers it → what the handler does → what the service does → where data lands → response. 3–6 sentences. No file paths.

---

## Guards (run all three before any work)

**Guard A** — if `wiki/index.md` already exists in the current working directory, **stop**:
> "A wiki already exists here (`wiki/index.md` found). Run `/repo-llm-wiki refresh` to regenerate."

**Guard B** — if `wiki/` exists and contains **wiki content** but no `wiki/index.md`, **stop**:
> "A `wiki/` directory exists with N wiki file(s) but no `index.md`. Move or remove it before running `/repo-llm-wiki init`."

"Wiki content" means: any `*.md` file anywhere under `wiki/` (other than `wiki/index.md` itself), OR any non-empty subdirectory other than the tool/meta dirs `.obsidian/`, `.archive/`, `.git/`. Pre-existing Obsidian vault config (`.obsidian/`), an empty `wiki/modules/`, or a pre-existing `wiki/.archive/` are **not** wiki content — proceed without stopping. If unsure, list what's present and ask before proceeding.

**Guard C** — verify we're inside a git working tree:
```bash
git rev-parse --is-inside-work-tree
```
If this returns anything other than `true`, **stop**:
> "repo-llm-wiki requires a git repository. Run `git init` here first."

---

## Step 1 — Repo identity and inventory (single agent, fast)

Print: *"Collecting repo data..."*

Run these directly. They are cheap and deterministic. Hold results in memory.

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

### 1.3 Directory tree (3 levels, for subagent context)

```bash
find . -type d -maxdepth 3 \
  -not -path '*/.git/*' \
  -not -path '*/node_modules/*' \
  -not -path '*/vendor/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*' \
  -not -path '*/archive/*' \
  | sort
# → DIR_TREE
```

### 1.4 Runtime versions (authoritative source, never from README)

Search candidate paths in order; **first match wins**. Leave `unknown` if no source exists.

| Variable | Candidate paths | Extract |
|---|---|---|
| `GO_VERSION`, `MODULE_NAME` | `go.mod`, `app/go.mod`, `service/go.mod` | `^go (\S+)` and `^module (\S+)` |
| `NODE_VERSION`, `NPM_NAME` | `package.json`, `app/package.json` | `.engines.node`, `.name` |
| `TF_VERSION` | `infra/terraform.tf`, `terraform.tf`, `infra/versions.tf`, `versions.tf` | `required_version = "..."` |
| `AWS_PROVIDER_VERSION` | same as TF_VERSION | first `aws { ... version = "..." }` under `required_providers` |

### 1.5 cmd/ entry-point enumeration

```bash
find ./app/cmd ./cmd -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort   # → CMD_DIRS
```

### 1.6 Infrastructure enumeration

```bash
find infra -maxdepth 1 -name '*.tf' 2>/dev/null | sort                                          # → TF_FILES
grep -rh "^resource \"aws_" infra/*.tf 2>/dev/null | grep -oE '"aws_[^"]+"' | sort | uniq -c    # → AWS_RESOURCE_COUNTS
find infra/environments -type f -name '*.tfvars' 2>/dev/null | sort                             # → ENV_FILES
```

`AWS_RESOURCE_TOTAL` = sum of counts in `AWS_RESOURCE_COUNTS`. Used for the infra-splitting decision in §3.

### 1.7 Scripts

```bash
find . -maxdepth 1 -name '*.sh' -type f | sort                          # → ROOT_SCRIPTS
test -f Makefile && echo Makefile                                       # → HAS_MAKEFILE
```

### 1.8 Workflows and tests

```bash
ls .github/workflows/ 2>/dev/null | grep -E '\.ya?ml$' | sort           # → WORKFLOW_FILES
find tests -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort           # → TEST_SUITES
```

### 1.9 Skip lists (do not read)

- `archive/`, `**/*_old*/`, `dist/`, `build/`, `vendor/`, `node_modules/`
- `*.csv`, `*.xlsx`, `*.xls`, `*.sql`, `*.dump`, `*.sum`, `*.lock`, `*.tfstate`
- `*_test.go`, `*.test.ts`
- Files larger than 1000 lines unless explicitly listed by a subagent task below

---

## Step 2 — Deep analysis (subagents in parallel)

Token-spend warning: the goal is to fan out so the main agent's context stays clean. Spawn the four subagents below **in a single message** (parallel) using the `Agent` tool with `subagent_type=general-purpose`. Wait for all to return before proceeding.

Each subagent prompt must be self-contained: it has no view of this conversation. Include `DIR_TREE`, the relevant lists from §1, and the exact return format.

### Subagent A — cmd/ flow narratives

**Sub-batching rule (run before spawning).** If `len(CMD_DIRS) > 20`, shard `CMD_DIRS` into batches of at most 15 entries each (sorted alphabetically) and spawn one Subagent A instance per shard, in parallel. This keeps each subagent's input bounded (~15 dirs × 3 files × 200 lines = ~9k lines) and protects against truncation on large repos. The main agent merges all shards' returns before §3. If `len(CMD_DIRS) ≤ 20`, run as a single subagent.

**Purpose**: for every entry in the assigned `CMD_DIRS` shard, classify it (Lambda / CLI tool / migration script / other) and produce a 3–6 sentence flow narrative.

**Per-entry investigation** (cap reads at 200 lines each):
1. Read `<dir>/main.go` (or first `*.go`).
2. Classify into **exactly two types**: contains `aws-lambda-go` import or `lambda.Start(` call → **Lambda**. Otherwise → **CLI tool**. There is no "Migration script" type — a migration runner that uses `lambda.Start` is a Lambda, one that has a plain `main()` is a CLI tool. Whether something is migration-related is captured by the `deprecated` flag and by naming, not by a separate category.
3. From the imports in main.go, identify the primary `internal/handler/<X>` and `internal/service/<Y>` packages used.
4. Read **one** handler file under that handler package and **one** service file under that service package (first `*.go` not ending `_test.go`), capped at 200 lines.
5. Note what the service does at a high level (writes to DynamoDB? calls another service? publishes to SQS? returns data?).

**Per-entry return** (one JSON-ish object):
```
{
  "name": "amendArrangement",
  "type": "Lambda",              // exactly one of: "Lambda", "CLI tool"
  "deprecated": false,           // true if dir name ends _old / _v1 / _legacy
  "trigger": "API Gateway",      // for Lambdas: "API Gateway" | "EventBridge schedule" | "SQS message" | "DynamoDB Streams" | "S3" | "Step Functions" | "unknown". For CLI tools: "CLI invocation".
  "flow": "Triggered by API Gateway. The handler parses and validates the amend-arrangement request, then calls the authorisation service which updates the existing arrangement record in DynamoDB. Returns the updated arrangement to the caller."
}
```

**No `category_hint` field.** Type drives placement: `Lambda` → `wiki/lambdas/`, `CLI tool` → `wiki/cli/`. Do not suggest other categories.

**Trigger inference rules** (subagent applies, in order):
- `aws-lambda-go/events.APIGatewayProxyRequest` in handler → `API Gateway`
- `events.CloudWatchEvent` / `events.EventBridgeEvent` → `EventBridge schedule`
- `events.SQSEvent` → `SQS message`
- `events.DynamoDBEvent` → `DynamoDB Streams`
- `events.S3Event` → `S3`
- Non-Lambda with `main()` parsing flags → `CLI invocation`
- Otherwise → `unknown`

**Return**: a single Markdown section per entry, plus a final summary table. Cap total return to ~6k tokens.

### Subagent B — Terraform resource inventory

**Purpose**: for each AWS resource type used in `infra/*.tf`, return type, count, up to 3 example resource names, and one-sentence purpose inferred from naming + the `.tf` file it lives in. Also identify GSI names from any `global_secondary_index` blocks (read `infra/dynamodb.tf` if present) and list environment files from `ENV_FILES`.

Inputs to pass: `TF_FILES`, `AWS_RESOURCE_COUNTS`, `ENV_FILES`.

**Return shape**:
```
{
  "resources_by_type": [
    {"type": "aws_lambda_function", "count": 42, "examples": ["create_authorisation","amend_arrangement","..."], "purpose": "Compute for the authorisation API and scheduled jobs"},
    {"type": "aws_dynamodb_table", "count": 1, "examples": ["authorisations"], "purpose": "Single-table store for authorisation records"},
    ...
  ],
  "dynamodb_gsis": ["GSI_CustomerAuthorisations", "GSI_ArrangementAuthorisations", ...],
  "environments": ["dev","sit","prod", ...]
}
```

Cap return ~4k tokens.

### Subagent C — Scripts grouping

**Purpose**: read each script in `ROOT_SCRIPTS` (first 50 lines is enough) and the Makefile if present. Group by name prefix (e.g. all `build-*.sh` together) and by purpose. For each group, write 1–2 sentences describing what the group does and list the scripts in it.

**Return shape**:
```
{
  "groups": [
    {"name": "Build scripts", "purpose": "Cross-compile Go Lambdas into deployment zips.", "scripts": ["build.sh","build-apimetrics.sh","build-dbconsentsummary.sh", ...]},
    {"name": "Data-tooling scripts", "purpose": "Run one-off data cleanups against DynamoDB.", "scripts": ["delete-dataholder.sh","list-dataholders.sh"]},
    ...
  ]
}
```

Singletons (one script in a group) are allowed but prefer grouping by prefix when ≥3 share one.

### Subagent D — Repo purpose paragraph

**Purpose**: write the 3–5 sentence overview paragraph that goes at the top of `index.md`. **Do not** take `README.md` at face value — it may be stale. Read:
- `README.md` (full, up to 500 lines)
- Names of all entries in `CMD_DIRS` (passed as input)
- `DIR_TREE` (passed as input)
- The top of one representative service file (first 100 lines), chosen by Subagent D from internal/service/.
- Up to 3 swagger/openapi files under `infra/swaggers/` (first 50 lines each) if they exist.

Synthesize what the repo actually *does* in plain language. Identify the domain (e.g. "Open-banking data-sharing authorisations under the Australian CDR regime"). State the deployment shape briefly (e.g. "Deployed as AWS Lambdas behind API Gateway, with DynamoDB as the system of record"). Do not invent capabilities; if uncertain, say so with `Confidence: low`.

**Return**: 3–5 sentences of prose, ready to drop into `index.md`. No headings, no citations, no wikilinks.

---

## Step 3 — Decide page layout (main agent)

After subagents return, the main agent decides which pages to emit. **Emit no empty sections, no empty pages.**

### 3.1 The four categories (fixed)

The category set is **closed**. The skill emits at most these four top-level pages, in this order on the index. Do not introduce others.

| Category | Page | Emit when |
|---|---|---|
| Lambdas | `wiki/lambdas.md` + `wiki/lambdas/<slug>.md` per item | at least one Subagent A item with `type=Lambda` |
| CLI tools | `wiki/cli.md` + `wiki/cli/<slug>.md` per item | at least one Subagent A item with `type=CLI tool` |
| Scripts | `wiki/scripts.md` | Subagent C returned at least one script group, **or** `WORKFLOW_FILES` is non-empty |
| Infrastructure | see §3.2 | `AWS_RESOURCE_TOTAL > 0` |

CI workflows live as a `## CI workflows` subsection inside `wiki/scripts.md` — not a category. Tests, if present, live as a `## Tests` paragraph inside `wiki/index.md` — also not a category.

**Items with `deprecated=true`** (e.g. `pfbToAMSMigration_old`, `basiqToPireanMigration_old`) stay in their natural category (`lambdas` if `type=Lambda`, `cli` otherwise) but are tagged `⚠ deprecated` in the row and in the detail page header. Do not segregate them into a separate page.

**No detail-page grouping rule.** Every Lambda gets its own detail page. Every CLI tool gets its own detail page. The v0.2.0 rule that allowed inlining ≥6 prefix-sharing items is removed — it was the source of the "lambdas hiding in migrations.md" failure.

### 3.1.1 Lambda-count consistency

The count in `lambdas.md`'s opening sentence and any count in the overview paragraph must equal `len({items where type=Lambda})`. If Subagent D's overview prose mentions a different count — typically because it counted Terraform `aws_lambda_function` modules — reconcile by trusting the Terraform count (from Subagent B) and updating both `lambdas.md` and the overview to match. If the discrepancy is real (e.g. cmd-based count says 23 but Terraform deploys 26 because 3 Lambdas are sourced from outside `cmd/`), state it once on `lambdas.md`: "23 Lambdas built from `cmd/`; Terraform deploys 26 — see [[infra/lambda-functions]]."

### 3.2 Infrastructure split rule

- If `AWS_RESOURCE_TOTAL ≤ 15` **or** there are ≤ 3 distinct resource types → emit a single `wiki/infra.md`.
- Otherwise → emit `wiki/infra.md` as a list of resource types (one per row, linking to a detail page) and emit `wiki/infra/<resource-type-slug>.md` for each type with **count ≥ 3**. Types with fewer than 3 resources are listed on `wiki/infra.md` directly without a detail page.

Slug rule for resource-type pages: strip `aws_` prefix and pluralise simply (`aws_lambda_function` → `lambdas`, `aws_dynamodb_table` → `dynamodb-tables`, `aws_s3_bucket` → `s3-buckets`). If a slug collides with a top-level category (`lambdas` for `aws_lambda_function`), prefix with `infra-` (→ `infra/infra-lambdas.md`) or, equivalently, keep it under `infra/` and the parent folder already disambiguates — use the latter (no prefix needed when nested under `infra/`).

---

## Step 4 — Create directories

```bash
mkdir -p wiki
# Create sub-directories only when their category will be emitted:
#   mkdir -p wiki/lambdas
#   mkdir -p wiki/cli
#   mkdir -p wiki/infra
```

---

## Step 5 — Write pages

For each page below, after writing it print: `wrote: wiki/<path>`. Skipped: `skipped: wiki/<path> (<reason>)`. Track `PAGES_WRITTEN_COUNT`.

### 5.1 Category list pages

For each emitted category, write its list page from the subagent return.

**`wiki/lambdas.md`** (template):
```markdown
# Lambdas

This repo deploys <N> AWS Lambda functions. Click into each for what it does.

| Lambda | Trigger | What it does (1 line) |
|---|---|---|
| [[lambdas/amendArrangement]] | API Gateway | Updates an existing arrangement |
| [[lambdas/createAuthorisation]] | API Gateway | Writes a new authorisation record |
| ... | | |
```

The "what it does (1 line)" column is the first sentence of the subagent's flow narrative, truncated to ~80 chars. Sort alphabetically.

**`wiki/cli.md`**: same structure as `lambdas.md`, with the `Trigger` column relabeled `Invocation`.

**Category pages are flat tables.** Do not introduce subsections grouping items by purpose, by deprecation, by pipeline stage, or any other axis. A single table per category page, sorted alphabetically. The `Type` (for Lambdas, this means trigger; for CLI tools, "CLI invocation") and `⚠ deprecated` tag in the name column carry all the categorisation a reader needs. Subsections are how the previous version drifted into inventing categories — they're banned here.

**`wiki/scripts.md`** (template):
```markdown
# Scripts

## <Group name from Subagent C>
<1–2 sentences from Subagent C>
- `build.sh` — <one-line purpose>
- `build-apimetrics.sh` — <one-line purpose>
...

## <Next group>
...

## CI workflows
<one line per WORKFLOW_FILES entry: file + inferred purpose>
```

### 5.2 Per-item detail pages

Write `wiki/lambdas/<name>.md` for every `type=Lambda` item and `wiki/cli/<name>.md` for every `type=CLI tool` item. **Every item gets a detail page** — no inline grouping, no shortcuts. **Slug = item name verbatim** (preserve case from `CMD_DIRS`).

**Hard rule, re-stated for the writer**: the body of a detail page contains **no relative paths to source files**. No `../app/...`, no `[handler.go](../...)`. Refer to handlers and services by their conceptual role ("the handler", "the authorisation service"), not by file path. Lint E2 fails the wiki otherwise. If the subagent's flow narrative contains such a path, strip it before writing.

Format:

```markdown
# <Name>

> Type: Lambda · Trigger: <trigger>{{ · ⚠ deprecated}}

<3–6 sentence flow narrative from Subagent A. No file paths. No wikilinks unless they point to other wiki pages.>

## Related
- [[lambdas]] — full list
{{- [[infra/dynamodb-tables]] — if the flow mentions DynamoDB and that page exists}}
{{- [[infra/sqs-queues]] — if the flow mentions SQS and that page exists}}
```

The `Related` section should list at most 3 wikilinks, only to pages that actually exist after layout decisions in §3.

For deprecated items (`deprecated: true` from Subagent A), prepend `⚠ deprecated · ` to the type line and add one sentence at the end: "This component appears deprecated based on its directory name; verify before relying on it."

### 5.3 Infrastructure pages

**Single-page case (`wiki/infra.md`)**:
```markdown
# Infrastructure

This repo's AWS infrastructure is defined in Terraform under `infra/`. Environments: <list from Subagent B>.

## Resources
| Type | Count | Examples |
|---|---|---|
| aws_lambda_function | 42 | create_authorisation, amend_arrangement, … |
| aws_dynamodb_table | 1 | authorisations |
| ... | | |

## DynamoDB
Single table `authorisations` with the following Global Secondary Indexes: <list>.

## Environments
<bulleted list>
```

**Split-page case**: `wiki/infra.md` becomes a router:
```markdown
# Infrastructure

This repo's AWS infrastructure is defined in Terraform under `infra/`. Environments: <list>.

## Resource types
- [[infra/lambdas]] — 42 Lambda functions
- [[infra/dynamodb-tables]] — 1 table with 7 GSIs
- [[infra/api-gateways]] — 1 REST API
- ...

## Small inventories
<types with count <3, listed inline with their resource names>

## Environments
<bulleted list>
```

Each `wiki/infra/<slug>.md` has:
```markdown
# <Resource type, human-readable plural>

There are <N> <resource type> in this repo, defined under `infra/`.

<1–2 sentences from Subagent B about what these resources do collectively.>

## Resources
<bulleted list of names — all of them>

{{ Section-specific notes, e.g. for DynamoDB: list GSIs; for Lambdas: link [[lambdas]] for runtime descriptions. }}
```

### 5.4 `wiki/log.md`

Read `<skill-dir>/templates/wiki/log.md.tmpl`. Substitute:
- `{{TIMESTAMP}}` → current UTC timestamp, `YYYY-MM-DD HH:MM UTC`
- `{{COMMAND}}` → `init`
- `{{SHA}}` → first 7 chars of `REPO_SHA`
- `{{BRANCH}}` → `REPO_BRANCH`
- `{{PAGE_COUNT}}` → write `__PAGE_COUNT__` for now; after `index.md` is written, count all `*.md` files inside `wiki/` recursively and `sed -i` (or equivalent) to replace `__PAGE_COUNT__`.
- `{{TOOL_VERSION}}` → `0.3.0`

### 5.5 `wiki/index.md` (last)

Compute staleness suffix: for `init`, always empty (`REPO_SHA == HEAD`).

Read `<skill-dir>/templates/wiki/index.md.tmpl`. Placeholders and their values:

- `{{REPO_NAME}}` → `REPO_NAME`
- `{{DATE}}` → today, `YYYY-MM-DD`
- `{{SHA}}` → first 7 chars of `REPO_SHA`
- `{{STALE_SUFFIX}}` → empty for `init`; `refresh` populates if applicable
- `{{OVERVIEW_PARAGRAPH}}` → the prose returned by Subagent D, dropped in verbatim
- `{{CATEGORY_LINKS}}` → one bullet per emitted category, in this fixed order: Lambdas, CLI tools, Scripts, Infrastructure. Format: `- [[lambdas]] — N Lambda functions`. Omit any category that wasn't emitted. Do not add bullets for categories outside this set.
- `{{TECH_STACK_SECTION}}` → if any of `GO_VERSION` / `NODE_VERSION` / `TF_VERSION` / `AWS_PROVIDER_VERSION` is not `unknown`: emit `\n## Tech stack\n\n<Markdown table>\n` with one row per known version (Runtime, Terraform, AWS provider). Skip unknown rows. If all four are `unknown`, substitute the empty string — the heading is part of the placeholder so it disappears with the section.
- `{{TESTS_SECTION}}` → if `TEST_SUITES` non-empty: `\n## Tests\n\n<one paragraph naming the test suite directories>\n`. Else empty string.

Never include a wikilink to a page that wasn't written.

Never include a wikilink to a page that wasn't written.

### 5.6 `AGENTS.md` (or extension)

Read `<skill-dir>/templates/AGENTS_BLOCK.md`. Call its content `BLOCK`.

- If `AGENTS.md` does not exist: write `BLOCK` as the entire file.
- If `AGENTS.md` exists and contains `<!-- repo-llm-wiki: begin -->`: leave untouched, print `wiki block already present in AGENTS.md`.
- Otherwise append a blank line then `BLOCK` to the end. **Never replace** existing content.

---

## Step 6 — Completion summary

```
Wiki generated in wiki/ (<PAGES_WRITTEN_COUNT> pages)

  wiki/index.md          ← start here
  wiki/lambdas.md        (+ <N> detail pages)
  wiki/cli.md            (+ <N> detail pages)
  wiki/scripts.md
  wiki/infra.md          (+ <N> detail pages, if split)
  wiki/log.md
  AGENTS.md              ← extended with wiki block

SHA: <REPO_SHA[0:7]>  Branch: <REPO_BRANCH>

Next: open wiki/index.md, then commit wiki/ to git.
Run /repo-llm-wiki lint to validate.
```

List only categories that were actually emitted.

---

## Atomicity note

Pages are written directly to `wiki/`. If a step fails mid-generation, the wiki is in a partial state. Recovery: remove `wiki/` and re-run `init`.
