# repo-llm-wiki — Phase 1 PRD

> Snapshot wiki: generate a committable Markdown wiki from a repository's current state.
> Sibling to [`pin-llm-wiki`](../../pin-llm-wiki/README.md) (which ingests external sources).
> Scope: Phase 1 only. Phases 2–4 are referenced for context, not designed here.

Status: draft 2 · Date: 2026-05-02 · Owner: nenad

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
| **New developer** | "I just joined this team. What is this repo?" | Reads `wiki/index.md` + `wiki/architecture.md` in 15 minutes; can locate the right module to look at next. |
| **Existing developer** | "I haven't touched this service in 9 months." | Skims `wiki/modules/<service>.md`; remembers the shape without re-reading code. |
| **AI coding agent** | Asked to debug or extend a feature. | Reads `AGENTS.md` → `wiki/index.md` → relevant module page before grepping; gets architectural context in ~2k tokens instead of ~50k. |
| **On-call / SRE** | "Something broke at 2am, what is this thing?" | `wiki/runbook.md` + `wiki/infra.md` give enough orientation to act. |
| **Auditor / security reviewer** | "What does this system handle?" | `wiki/overview.md` + glossary explains domain and data classes in plain language. |

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

```
wiki/
  index.md              # landing page; TOC; "read me first"; links to every other page
  overview.md           # what this repo is, in 200–400 words, audience-tagged
  architecture.md       # components, data flow, external dependencies, diagrams (mermaid)
  modules/              # one page per top-level module / service / package
    <module>.md
  infra.md              # IaC repos only: stacks, environments, deployment modes
  runbook.md            # build / run / deploy / debug; commands lifted from scripts + README
  repo-map.md           # honest tour of the file tree, including messy parts
  glossary.md           # domain terms with definitions, code citations
  log.md                # append-only generation history (when, version, repo SHA)
  .archive/             # soft-deleted pages

AGENTS.md               # created or extended; tells agents to read wiki/index.md first
```

Pages omitted when not applicable. A pure-library repo gets no `infra.md`; a static-site repo may get no `modules/` if there's only one module. The skill decides this from the repo profile.

### 5.3 Repo-type awareness

The skill detects repo profile and adjusts emphasis. Profile detection is heuristic, based on top-level files:

| Detected profile | Signals | Emphasis |
|---|---|---|
| `iac-aws` | `*.tf`, `infra/`, `serverless.yml` | `infra.md` foregrounded; envs, deploy modes, IAM, resource inventory |
| `go-lambda-monorepo` | `go.mod` + many `cmd/*/main.go` | enumerate Lambda set, group by domain; surface entrypoint count |
| `node-service` | `package.json` with server deps | runtime, routes, env config |
| `library` | `package.json`/`go.mod` without bin/server | API surface, public exports |
| `frontend-app` | `package.json` + React/Vue/Svelte | route map, component tree, build pipeline |
| `mixed` / `unknown` | fallback | generic template, lower confidence flag |

Phase 1 ships with **2 profiles**: `iac-aws` and `generic`. Others are stubs to be filled in later. We pick `iac-aws` first because it's the target validation repo's shape and the most-different-from-generic case.

### 5.4 No config file in Phase 1

All defaults — which directories to skip, how to detect profile, what to flag as sensitive — are **embedded in the skill definition** (`SKILL.md` and its workflow files). No `.repo-llm-wiki.yml` is created. This keeps the setup to a single command with no mandatory configuration step.

The profile can be overridden by passing it as an argument: `/repo-llm-wiki init iac-aws`. Without an argument, the skill auto-detects it.

### 5.5 Citations and accuracy rules

Hard rules, enforced by `lint`:

1. **Every architectural claim cites a file.** Format: `([app/internal/service/authorisation/service.go](../app/internal/service/authorisation/service.go))`. No claim → lint warns.
2. **Every named component exists.** If `wiki/architecture.md` says "X service," the file/dir for X must resolve. If it doesn't, lint errors.
3. **Counts are computed, not estimated.** "42 Lambda entrypoints" — verified by directory enumeration, not LLM guess.
4. **Versions are read from source.** Go version from `go.mod`, Node from `package.json` engines, Terraform from `terraform.tf`. Never from README prose.
5. **External services are listed only with evidence.** "Uses SQS" requires a Terraform resource or SDK import.
6. **Honesty markers.** When uncertain, page says so explicitly: `> Confidence: low — inferred from directory names; no clear entrypoint found.`

This is the lesson from the DeepWiki review: a confident wrong wiki is the worst outcome. Better a smaller wiki with clear confidence levels.

### 5.6 The "honest repo map" page

`wiki/repo-map.md` is a Phase 1 differentiator. It contains:

- The tree as it actually is, including the ugly parts.
- A **classification** for each top-level entry: `code` / `infra` / `tests` / `docs` / `scripts` / `data` / `archive` / `unclear`.
- A `**⚠ Loose files at root**` callout listing root-level files that don't fit the project structure (e.g. `IdPermFinal 1.java`, `consents-12.json`).
- A `**⚠ Likely deprecated**` callout for `*_old/`, `archive/`, anything matching configurable patterns.
- A `**⚠ Possibly sensitive**` callout for files with names suggesting customer data (`*customer*.csv`, `*consents*.json`) — does not read contents, just flags.

This page is the antidote to the "DeepWiki shows a clean idealised view" failure mode.

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
3. ✅ The wiki is **strictly better than DeepWiki** on at least these axes: correct Go version, complete workflow list, accurate Lambda count, presence of `repo-map.md` flagging messy reality, no claims that fail lint.
4. ✅ A teammate who has never seen the repo can answer 5 of 5 "where is X?" questions using only the wiki, in under 10 minutes.
5. ✅ A Claude Code agent told to use the wiki uses ≥50% fewer tokens to answer "what does this repo do and how is it deployed?" vs. cold (no-wiki) baseline.
6. ✅ Re-running `refresh` on an unchanged repo produces a diff of zero lines for non-LLM-generated pages (`repo-map.md`, `index.md` TOC, citation links) and ≤20% line churn on prose pages.
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
| 1 | Single agent or multi-agent? | **Single agent** — simpler, cheaper, easier to debug for Phase 1. |
| 2 | Mermaid diagrams? | **Yes, generated, marked as draft.** `lint` validates syntax only; human reviews arrows. |
| 3 | Wikilink flavour? | **`[[wikilinks]]`** — same as `pin-llm-wiki`; Obsidian-compatible; GitHub renders poorly but that's accepted. |
| 4 | `AGENTS.md` merge? | **Append**, inside a clearly delimited block. Never clobber an existing file. |
| 5 | Skill name? | **`/repo-llm-wiki`** — full name, no alias. |

## 11. Out of this PRD

Things that *will* exist in the implementation plan, deliberately not here:

- File layout of the skill itself (`skills/repo-llm-wiki/...`)
- The exact instructions at each stage (what the skill tells the agent to do)
- The token-budgeting strategy for large repos
- Page generation order and dependency sequencing
- Test harness structure
- Release / packaging steps

These belong in `spec/phase-1-implementation.md`.
