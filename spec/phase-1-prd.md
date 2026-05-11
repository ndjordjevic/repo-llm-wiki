# repo-llm-wiki — Phase 1 PRD

> Snapshot wiki: generate a committable Markdown wiki from a repository's current state.
> Sibling to [`pin-llm-wiki`](../../pin-llm-wiki/README.md) (which ingests external sources).
> Scope: Phase 1 only. Phases 2–4 are referenced for context, not designed here.

Status: draft 3 · Date: 2026-05-11 · Owner: nenad

> **Draft-3 reset.** Draft 2 produced a v0.1 wiki on the validation repo and the result was too heavy: a `repo-map`, a `glossary`, an `architecture` page with a useless 40-node mermaid, and module clusters that lumped 8 lambdas into one page. Draft 3 pivots to a **drill-down** wiki: `index.md` → category page → per-item flow narrative. Source-file citations are dropped (broken in Obsidian); architecture/repo-map/glossary/overview pages are dropped. Analysis fans out across subagents. See §5 and §10 for the changes.

---

## 1. Problem

Engineers joining a non-trivial repo spend days reverse-engineering its shape from code, scattered READMEs, and tribal knowledge. AI coding agents (Claude Code, Cursor, Copilot) face a related problem on every session: they can grep the repo, but they re-derive the same architecture summary token-by-token each time, and they have no source for the *why* (rationale, deprecated paths, operational constraints).

Hosted tools like DeepWiki produce a serviceable summary for public repos but are unusable for private corporate repos. Existing in-repo generators (Autodoc, etc.) produce file-level docs that are too granular to navigate.

**We need:** a tool that generates a small, accurate, committable wiki *inside* a repo, written at the altitude humans and agents actually use.

## 2. Goals

1. **One command, one wiki.** Run a slash command in Claude Code (or equivalent) inside a repo, get a populated `wiki/` directory.
2. **Accurate.** Every claim is either obvious from the file structure or cited back to a source file. No hallucinated services, no invented endpoints.
3. **Committable.** Output is plain Markdown, diff-reviewable, lives at the repo root. No hosting, no backend.
4. **Agent-readable.** A generated `AGENTS.md` (or block appended to an existing one) directs agents to read `wiki/index.md` first.
5. **Repo-type-aware.** A Terraform/AWS/Lambda repo gets different emphasis than a React app or a CLI tool.
6. **Honest about reality.** Surfaces messiness — archive/, `_old/` dirs, root-level scripts, large data fixtures — instead of producing a clean idealised view that misleads readers.

## 3. Non-goals (Phase 1)

- ❌ Auto-update on PRs / commits. *(Phase 2)*
- ❌ Jira / Confluence ingestion. *(Phase 3)*
- ❌ Cross-repo / wiki-of-wikis. *(Phase 4)*
- ❌ Code-graph / GraphRAG indexing. *(Out of scope for the project.)*
- ❌ Per-function or per-file docstrings. *(Different tool.)*
- ❌ A web UI, a hosted service, a search backend.
- ❌ Replacing Confluence / ADRs / READMEs that already exist. The wiki *links to* them.

## 4. Users and use cases

| User | Use case | Success looks like |
|---|---|---|
| **New developer** | "I just joined this team. What is this repo?" | Reads the overview paragraph in `wiki/index.md`, then drills into the category that matches their question — 10 minutes to a working mental model. |
| **Existing developer** | "I haven't touched this service in 9 months." | Opens the relevant detail page (e.g. `lambdas/amendArrangement.md`); reads 3–6 sentences and remembers the flow. |
| **AI coding agent** | Asked to debug or extend a feature. | Reads `AGENTS.md` → `wiki/index.md` → relevant category → relevant detail page before grepping. Detail page identifies which handler/service to look at by name, agent greps the code from there. |
| **On-call / SRE** | "Something broke at 2am, what is this thing?" | `wiki/index.md` + the lambda detail page for the offending function give enough orientation to act. |

Out-of-scope user: end customers / external API consumers. The wiki is internal.

## 5. Scope: what Phase 1 does

### 5.1 The slash command

```
/repo-llm-wiki init           # scaffold + first generation
/repo-llm-wiki refresh        # regenerate from current HEAD (still one-shot, not diff-aware)
/repo-llm-wiki lint           # validate links, citations, freshness
```

`refresh` is one-shot regeneration — Phase 1 does not attempt incremental updates. Calling it twice produces a (mostly) deterministic result; the only diff between runs should be from real repo changes or LLM nondeterminism.

### 5.2 What gets created

The wiki is a **drill-down**. From `index.md` a reader picks a category, lands on a list, clicks an item, and reads a short flow narrative.

```
wiki/
  index.md                  # overview paragraph + category links + tech stack
  lambdas.md                # list of Lambda entrypoints (if any)
  lambdas/<name>.md         # per-Lambda flow narrative (trigger → handler → service → store)
  cli.md                    # list of CLI tools (if any)
  cli/<name>.md
  migrations.md             # list of migration scripts (if any); often grouped, no per-item pages
  migrations/<name>.md      # only when an item warrants its own page
  scripts.md                # build + utility scripts, grouped by purpose; CI workflows section
  infra.md                  # IaC repos: resources, environments, DynamoDB summary, …
  infra/<resource-type>.md  # split out per resource type when total > 15 or types > 3
  log.md                    # append-only generation history
  .archive/                 # orphans from prior refresh runs

AGENTS.md                   # created or extended; tells agents to read wiki/index.md first
```

**What is deliberately not generated**:
- No `overview.md` — overview is the opening paragraph of `index.md`.
- No `architecture.md` — components are visible by browsing the categories; cross-cutting flow can be added later if a real reader needs it.
- No `repo-map.md` — the category index already shows the repo's shape; the "honest reality" goal is preserved by surfacing categories like *Migration scripts* and *Build scripts* rather than hiding them.
- No `glossary.md` — v0.1 generated too few terms to justify the page; deferred.
- No mermaid diagrams — v0.1 produced a 40-node flowchart that nobody could read; if a diagram is needed later, draw it by hand.

Categories are **discovered, not fixed**. A pure-library repo emits no `lambdas.md`. A repo with no Terraform emits no `infra.md`. The infra split rule: if `AWS_RESOURCE_TOTAL ≤ 15` or distinct types ≤ 3 → one `infra.md`; otherwise split per resource type (one page per type with count ≥ 3).

### 5.2.1 Detail-page contract

A detail page (e.g. `wiki/lambdas/amendArrangement.md`) contains:
- A one-line metadata header (`Type · Trigger`, optionally `⚠ deprecated`).
- **3–6 sentences of flow narrative**: who triggers the component, what the handler does, what the service does, where data lands, what the response is. Written so a reader gets a clear picture in 2 minutes.
- A `Related` section with at most 3 wikilinks to other wiki pages.

**Not in a detail page**: request/response schemas, file paths, function names, line numbers, code blocks. For those, the reader reads the source. The wiki sits one altitude above the code on purpose.

### 5.2.2 No source-file citations

The v0.1 wiki cited every claim with a relative path to a source file (`../app/internal/...`). In Obsidian — the primary reading environment — `wiki/` opens as the vault root, and those paths resolve to nothing: every link is broken. v0.2 uses **wikilinks between wiki pages only**. Source-file claims are still verifiable because every named component must exist (see §5.5 rule 2).

### 5.3 Repo-type awareness — via category discovery, not profiles

v0.1 used a small set of fixed profiles (`iac-aws`, `generic`) to switch templates. v0.2 drops that in favour of **category discovery**: the skill enumerates `cmd/`, `infra/`, scripts, and workflows independently, and emits a category only when there's evidence for it.

- Repo with `*.tf` files → `infra.md` (split if many resource types).
- Repo with `cmd/*/main.go` directories → `lambdas.md` and/or `cli.md` based on imports (`aws-lambda-go` presence).
- Repo with root `*.sh` or Makefile → `scripts.md`.
- Repo with `tests/` directories → tests paragraph in `index.md`.
- A pure-library repo with none of the above gets just an `index.md` overview — that's the honest minimum.

The benefit over profile detection: the skill works correctly on mixed repos (an `iac-aws` repo that *also* has CLI tools, a Go service that *also* has Terraform) without picking the wrong template. The cost: per-category prose conventions are bound up in the category contracts in `init.md` Step 5 rather than centralised. That trade-off is intentional for v0.2.

### 5.4 No config file in Phase 1

All defaults — which directories to skip, how to detect profile, what to flag as sensitive — are **embedded in the skill definition** (`SKILL.md` and its workflow files). No `.repo-llm-wiki.yml` is created. This keeps the setup to a single command with no mandatory configuration step.

The profile can be overridden by passing it as an argument: `/repo-llm-wiki init iac-aws`. Without an argument, the skill auto-detects it.

### 5.5 Accuracy rules

Hard rules, enforced by `lint`:

1. **No source-file citations.** Reversed from v0.1. Lint errors on any relative path leaving `wiki/`.
2. **Every named component exists in the repo.** If a detail page is written for `amendArrangement`, the directory `app/cmd/amendArrangement/` (or equivalent) must exist. Verified at generation time by Subagent A — there is no fabricated component because the page only exists when the cmd directory does.
3. **Counts are computed, not estimated.** Lambda count, resource counts, GSI counts — all from shell enumeration in Step 1.
4. **Versions are read from source.** Go from `go.mod`, Node from `package.json` engines, Terraform from `terraform.tf`. Never from README prose.
5. **External services are mentioned only with evidence.** "Writes to DynamoDB" requires either a Terraform `aws_dynamodb_table` or a relevant SDK import in the service file the subagent read.
6. **Honesty markers.** When the subagent's trigger inference returns `unknown`, the detail page says so. When the overview agent is uncertain about the domain, it prepends `> Confidence: low — …`.

### 5.6 Surfacing reality (without a dedicated page)

v0.1 had a `repo-map.md` page that flagged messy reality. v0.2 distributes that signal into the categories themselves:

- **Migration scripts** category: deprecated migration runners (`pfbToAMSMigration_old`, `basiqToPireanMigration_old`) appear in the list with a `⚠ deprecated` mark, not buried under "archive".
- **Scripts** category: groups all root-level `build-*.sh`, `delete-*.sh`, `list-*.sh` etc. so they're explicit, not invisible.
- **CLI tools** vs **Lambdas**: separated by import-level evidence, so a CLI helper accidentally living in `cmd/` doesn't get counted as a Lambda.
- **Confidence: low** markers on items where trigger or purpose couldn't be inferred.

What v0.1 also did via `repo-map` — flagging loose root files and possibly-sensitive data dumps — is **deferred** to phase 2 unless a real reader misses it. Surfacing it well requires more design than v0.2 has room for.

## 6. Distribution and runtime

- Distributed as a [skills.sh](https://skills.sh)-installable skill, same channel as `pin-llm-wiki`:
  ```bash
  npx skills@latest add ndjordjevic/repo-llm-wiki
  ```
- Runs in **Claude Code** (primary), **Cursor**, and **GitHub Copilot** via the same `SKILL.md` contract.
- No backend. No telemetry. No external services beyond the LLM the host agent already uses.
- Works on private repos by construction (it's just files and an LLM session you already have).

## 7. Success criteria

Phase 1 ships when, on the validation repo (`tfaws-dregdata-dhs-app-authorisation`):

1. ✅ `/repo-llm-wiki init` produces a `wiki/` in under 10 minutes wall-clock.
2. ✅ The wiki is **at least as accurate** as the DeepWiki baseline on the facts we already verified (single-table DynamoDB, 7 GSIs, 4-stage migration pipeline, hexagonal layout, lambda set).
3. ✅ The wiki is **strictly better than DeepWiki** on at least these axes: correct Go version, complete workflow list, accurate Lambda count, messy reality surfaced via the *Migration scripts* and *Scripts* categories (deprecated runners visible, build scripts grouped), no claims that fail lint.
4. ✅ A teammate who has never seen the repo can answer 5 of 5 "where is X?" questions using only the wiki, in under 10 minutes.
5. ✅ A Claude Code agent told to use the wiki uses ≥50% fewer tokens to answer "what does this repo do and how is it deployed?" vs. cold (no-wiki) baseline.
6. ✅ Re-running `refresh` on an unchanged repo produces a diff of zero lines for the `index.md` TOC, category list pages, and `infra.md` resource tables; ≤20% line churn on prose detail pages (`lambdas/<name>.md` etc.).
7. ✅ `lint` passes with zero errors on the generated output.

If any of 1–4 fails on the validation repo, Phase 1 isn't done.

## 8. Out-of-scope failure modes (acknowledged, not solved)

- **LLM nondeterminism in prose.** Two `refresh` runs will produce slightly different `overview.md`. Phase 1 accepts this; Phase 2's diff-driven update reduces it.
- **Token budget on huge repos.** Hierarchical summarization is in scope; running on a 10,000-file monorepo is not. Phase 1 targets repos up to ~2,000 files / ~500k LOC.
- **Quality on rare profiles.** A Rust/embedded/COBOL repo will get the `generic` template. That's expected.
- **Wiki rot if `refresh` is never re-run.** Phase 1 mitigates by stamping `log.md` with the SHA at last refresh and warning in `index.md` if HEAD has moved more than N commits since. Real fix is Phase 2.

## 9. Validation plan

Single validation target: `tfaws-dregdata-dhs-app-authorisation`.

- **Baseline already collected**: see `research/00-research-notes.md` and the DeepWiki comparison performed in conversation. Specific facts to beat: Go version (1.24, not 1.23), 9 workflows (not 5), 42 Lambdas (not unstated), and surfacing the root-level mess.
- **Acceptance test**: a teammate-style read-through against §7 criteria.
- **Stretch**: run on one additional repo of a different profile to confirm the `generic` fallback isn't broken.

## 10. Decisions (resolved)

These were open during drafting; all closed before implementation plan is written.

| # | Question | Decision |
|---|---|---|
| 1 | Single agent or multi-agent? | **Multi-agent fan-out for analysis, single writer for pages.** v0.1 went single-agent; on a 42-Lambda repo the main context got crowded enough that grouping decisions suffered. v0.2 spawns four subagents in parallel (cmd-flow, terraform, scripts, overview) and the main agent assembles their returns into pages. Style stays consistent because one agent writes. Hosts without a subagent primitive degrade to inline analysis — same output, slower. |
| 2 | Mermaid diagrams? | **No.** v0.1 generated a 40-node `architecture.md` flowchart and it was unreadable. Deferred until there's a clear use case. |
| 3 | Wikilink flavour? | **`[[wikilinks]]`** — Obsidian-compatible. GitHub renders them poorly; that's accepted. |
| 4 | Source-file citations? | **None in v0.2.** Obsidian opens `wiki/` as the vault root, so relative paths to source files (`../app/...`) resolve to nothing — every link was broken. Lint now errors on any relative path leaving `wiki/`. |
| 5 | `AGENTS.md` merge? | **Append**, inside a clearly delimited block. Never clobber an existing file. |
| 6 | Skill name? | **`/repo-llm-wiki`** — full name, no alias. |

## 11. Out of this PRD

Things that *will* exist in the implementation plan, deliberately not here:

- File layout of the skill itself (`skills/repo-llm-wiki/...`)
- The exact instructions at each stage (what the skill tells the agent to do)
- The token-budgeting strategy for large repos
- Page generation order and dependency sequencing
- Test harness structure
- Release / packaging steps

These belong in `spec/phase-1-implementation.md`.
