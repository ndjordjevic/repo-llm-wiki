# repo-llm-wiki — Research Notes

> Brainstorming doc. Not a PRD, not a plan. Captures prior art, open questions, and rough shape of the idea so we can decide what's worth building.

Date: 2026-05-02
Author: nenad (with Claude)

## 1. The idea in one paragraph

Build a Claude Code / Cursor / Copilot **skill** (sibling to [`pin-llm-wiki`](../../pin-llm-wiki/README.md)) that takes a software repo and emits a **local Markdown wiki** describing what's in it: architecture, modules, infra, runbooks, glossary, key flows. The wiki lives **inside the repo** (e.g. `wiki/`), is committed to git, and is consumed by:

1. **Humans** — onboarding new devs, refreshing existing devs, giving PMs/SREs a readable map of the system.
2. **AI agents** — pulled into context before debugging, feature work, or refactors so the agent reasons over distilled architecture rather than re-deriving it from raw files each session.

`pin-llm-wiki` ingests **external sources** (URLs, GitHub repos you don't own, docs, YouTube). `repo-llm-wiki` ingests **one source: the repo it lives in**, plus eventually its surrounding ecosystem (Jira, Confluence, sibling repos).

## 2. Phasing (user's framing, lightly sharpened)

| Phase | Scope | Trigger | Output |
|---|---|---|---|
| **1. Snapshot wiki** | One-shot. Generate wiki from current repo HEAD. | `/repo-llm-wiki init` | `wiki/` committed to repo |
| **2. Living wiki** | Wiki updates as devs land PRs / commit / close stories. | Git hook, GitHub Action, or post-merge agent | Diff-aware wiki updates, also via PR |
| **3. Org context** | Pull Jira tickets + Confluence pages into the wiki so domain rationale lives next to code. | On-demand or scheduled | `wiki/jira/`, `wiki/confluence/`, cross-links to `wiki/sources/` |
| **4. Wiki-of-wikis** | Multiple repos in an org → meta-wiki that indexes each repo's wiki, cross-references shared concepts (services, contracts, domain terms). | Central tool that crawls each repo's `wiki/` | Org-level index + cross-repo glossary |

## 3. The honest question: do AI agents even need this?

The user raised this directly. Worth taking seriously, because it shapes whether Phase 1 is for humans, agents, or both.

**The case it's redundant for agents:**
- Modern agents (Claude Code, Cursor, Copilot, Devin) already index the repo. They can grep/AST-walk on demand.
- A stale wiki is *worse* than no wiki — it confidently misleads.
- Agents excel at "just look at the code." Wiki is a distillation of code; the source is right there.

**The case it's not redundant:**
- **Distillation is not free.** Re-deriving "what does this service do, who calls it, what are the invariants" from 200 files every session burns tokens and time. A pre-computed summary at the right altitude is cheaper context than raw code.
- **Code answers "what," not "why."** Decisions, tradeoffs, deprecated paths, "we tried X and it didn't work" — these are not in the code. They're in commits, PRs, Jira, Slack, people's heads. The wiki is where they get pinned.
- **Cross-repo context.** Indexing tools work great inside one repo. They struggle across 12 repos that talk to each other via REST/Kafka/SQS. A meta-wiki (Phase 4) is genuinely new context, not redistilled context.
- **Humans need it regardless.** New devs onboarding, on-call engineers at 2am, auditors. The agent value is a bonus on top of the human value.
- **Agents-as-readers shape what we write.** AGENTS.md + wikilinks + structured pages = agents can navigate it cheaply. A Confluence page full of screenshots can't be navigated by an agent.

**My read:** Phase 1's primary value is **human onboarding** + **a stable pointer for agents** ("read `wiki/architecture.md` first"). The agent-side value gets stronger at Phases 3 and 4, where the wiki contains things that aren't recoverable from code alone (Jira "why," cross-repo "where").

This is also the position [Karpathy's LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) takes: the wiki is a *maintained memory layer*, not a regenerated cache.

## 4. Prior art (what already exists)

A non-trivial amount. We are not first. That's good — it means the problem is real and we can borrow patterns. The question is whether there's a useful seam none of them sit in.

### 4.1 DeepWiki (Cognition / Devin)

- **What:** Replace `github.com` with `deepwiki.com` → AI-generated wiki with diagrams, Q&A chat, cross-references.
- **Hosted.** They've indexed 50k+ public repos. Private repos via auth.
- **Strength:** Frictionless, polished, mature. Devin's research agent does the heavy lifting.
- **Gap vs. our idea:**
  - Lives **outside** the repo (their domain, not your `wiki/` folder).
  - Not committable, not diff-reviewable, not local.
  - Refresh cadence is theirs, not yours.
  - Doesn't compose with Jira/Confluence in the way Phase 3 imagines.
- **Open-source clones:** [`deepwiki-open` (AsyncFuncAI)](https://github.com/AsyncFuncAI/deepwiki-open), [`OpenDeepWiki` (AIDotNet, C#/TS)](https://github.com/AIDotNet/OpenDeepWiki), [`deepwiki-rs` / Litho (sopaco)](https://github.com/sopaco/deepwiki-rs) — these address the "I want it self-hosted" gap but are still standalone webapps, not in-repo skills.
- **Devin's official repo:** [CognitionAI/deepwiki](https://github.com/CognitionAI/deepwiki).

### 4.2 Karpathy's LLM Wiki pattern

- **What:** Manually-curated `wiki/` of `.md` files, fed to LLM as context. [Original gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).
- **Strength:** Dirt simple, durable, git-native, agent-friendly.
- **Weakness:** No automation — Karpathy curates by hand.
- **`pin-llm-wiki` already automates the *external-source* version of this.** `repo-llm-wiki` would automate the *self-source* version.

### 4.3 Autodoc (context-labs)

- [GitHub](https://github.com/context-labs/autodoc). DFS over the repo, LLM writes per-file/per-folder docs into `.autodoc/`.
- **Strength:** First mover (2023), simple model.
- **Weakness:** File-level docs ≠ architecture-level wiki. Output is granular and noisy. Dormant repo.

### 4.4 DocAgent (research, [arXiv 2504.08725](https://arxiv.org/html/2504.08725v1))

- Multi-agent system: Reader → Searcher → Writer → Verifier → Orchestrator. Topological code traversal so each doc is generated with prerequisites already documented.
- **Strength:** Addresses the "you can't write a doc for a function until you've documented its dependencies" ordering problem. Worth borrowing.
- **Weakness:** Academic; not a product.

### 4.5 CodeWiki (open-source framework)

- [Reference write-up](https://muhammadraza.me/2026/building-codewiki-compiling-codebases-into-living-wikis/). Hierarchical multi-agent: master orchestrator delegates to specialized module agents.
- Closer to the *shape* of what we'd build. Worth reading deeply before committing to an architecture.

### 4.6 ai-doc-gen (divar-ir)

- [GitHub](https://github.com/divar-ir/ai-doc-gen). Multi-agent, GitLab integration, concurrent processing.
- Good reference for "scale to large codebase without serial LLM calls."

### 4.7 Mintlify, Docsie, Kodesage

- Commercial. Mintlify = AI-native docs platform with `llms.txt` + MCP server. Docsie = AI agent that surfaces docs inside Jira. Kodesage = ties code + tickets + DB schema + offline docs into a queryable knowledge base, marketed for legacy systems.
- **Relevance:** Phase 3/4 territory. They're already doing the Jira-Confluence-Code fusion, but as SaaS. Our equivalent is repo-local + open.

### 4.8 Knowledge-graph approaches (GraphRAG over code)

- [GitNexus](https://github.com/abhigyanpatwari/GitNexus), [Memgraph Graph-Code](https://memgraph.com/blog/graphrag-for-devs-coding-assistant).
- They build a graph of symbols/calls/deps, expose it to agents via MCP/RAG. Strictly more powerful than markdown for "find all callers of X across services."
- **But:** lossy on intent and rationale. A graph tells you "function A calls B," not "we route through B because legal said so in 2024."
- Probably *complementary* to a markdown wiki, not a replacement. Markdown for narrative, graph for structure.

### 4.9 Per-repo wiki via GitHub Actions

- [earezki.com 2026-04 article](https://earezki.com/ai-news/2026-04-24-building-a-per-repo-wiki-that-actually-gets-read/) — concrete pattern: GitHub Action runs on merge, regenerates affected wiki pages, opens a PR.
- Closest blueprint for our **Phase 2**.

### 4.10 AGENTS.md as a convention

- Growing standard. [Builder.io explainer](https://www.builder.io/blog/agents-md). `pin-llm-wiki` already emits one.
- Phase 1 should emit an `AGENTS.md` (or extend an existing one) pointing at `wiki/index.md` so agents auto-consult it.

## 5. Where does `repo-llm-wiki` fit, then?

Mapping ourselves against prior art, the seams worth claiming:

1. **In-repo, git-reviewable, agent-readable.** DeepWiki is hosted; Mintlify is hosted; Autodoc is in-repo but file-level not architecture-level. `repo-llm-wiki`'s shape — committed `wiki/` of curated narrative pages with `[[wikilinks]]` and `AGENTS.md` — is exactly the `pin-llm-wiki` shape, applied inwards. That's a defensible niche.
2. **Skill-shaped, not platform-shaped.** Installs via `npx skills@latest add ...`, runs in Claude Code / Cursor / Copilot. No backend, no dashboard, no auth wall. Same delivery model as `pin-llm-wiki` — that's the lane.
3. **Phased into Jira/Confluence/multi-repo.** Most existing tools stop at "wiki for one repo." Phases 3 and 4 are the differentiators, and they're what enterprise users actually need.
4. **Diff-aware updates (Phase 2).** Refresh wiki pages from the PR diff, not by re-running the whole pipeline. Cheaper, reviewable, non-destructive.

Things to consciously **not** try to be:
- Not a hosted service. (DeepWiki wins.)
- Not a code-graph / GraphRAG engine. (GitNexus, Memgraph win — and we can integrate later.)
- Not a Confluence replacement. (Phase 3 ingests *from* Confluence; doesn't try to host pages.)
- Not a per-function docstring generator. (doc-comments-ai wins.)

## 6. Open design questions for Phase 1

Captured, not resolved.

### 6.1 What pages does the snapshot wiki contain?

Strawman:
- `wiki/index.md` — landing page, TOC, "read me first"
- `wiki/overview.md` — what does this repo do, in 200 words, with audience tags (dev/SRE/PM)
- `wiki/architecture.md` — components, data flow, external dependencies
- `wiki/modules/<name>.md` — one per top-level module/package/service
- `wiki/infra.md` — for IaC repos: stacks, environments, secrets handling
- `wiki/runbook.md` — how to build, run, deploy, debug
- `wiki/glossary.md` — domain terms with definitions and code links
- `wiki/decisions.md` — ADR-style; seeded from commit/PR history if available
- `AGENTS.md` — agent reading order

For the target repo (`tfaws-dregdata-dhs-app-authorisation`, a Terraform/AWS auth project with mixed Java/Python/shell), `wiki/infra.md` and `wiki/runbook.md` matter more than `wiki/modules/`. Repo-type-aware templates feel like the right abstraction.

### 6.2 How do we keep it from being LLM slop?

- **Cite back to source.** Every claim links to a file/line, mirroring how `pin-llm-wiki` cites `raw/`.
- **Detail levels** (`brief` / `standard` / `deep`) like `pin-llm-wiki` already has.
- **Lint phase** — broken wikilinks, missing citations, contradictions.
- **Bias to less.** A 6-page wiki that's right beats a 60-page wiki with hallucinations.

### 6.3 Token budget on big repos

- The target repo has Java + Python + shell + Terraform + tests + archived migration scripts. Probably hundreds of files.
- Can't put it all in one prompt. Strategy options:
  - **Hierarchical summarization** (CodeWiki / DocAgent style): summarize files → modules → services → repo.
  - **Map-reduce over the dependency graph.**
  - **Skip lists**: `archive/`, `tests/fixtures/`, generated code, vendored deps. (Big win on this repo specifically — half the root is `build-*.sh` scripts and `*_old` migrations.)
- Worth treating as a real engineering problem, not an afterthought.

### 6.4 Is the wiki opinionated about format?

`pin-llm-wiki` says: Markdown + `[[wikilinks]]` + Obsidian-friendly. That's a good default. But:
- GitHub renders `[[wikilinks]]` poorly outside the GitHub Wiki feature.
- VS Code / Cursor / Obsidian / Foam handle it fine.
- Decide: do we lean Obsidian (best for humans navigating locally) or GitHub-flavored (best for casual readers on github.com)? Probably Obsidian-style with a `lint` rule that also emits a flat `wiki/_github-toc.md` for browser readers.

## 7. Open design questions for Phases 2–4

### 7.1 Phase 2: Living wiki

- **Trigger:** post-merge GitHub Action vs. local pre-commit vs. Claude Code agent that re-runs on demand. Probably all three, gated by config.
- **Diff-driven update:** for a PR touching `app/auth/handler.py`, only refresh `wiki/modules/auth.md`, `wiki/architecture.md` (if public surface changed), and append to `wiki/decisions.md` if PR description carries an ADR marker.
- **Author signal:** mine PR title / description / linked Jira / commit messages for the *why*. This is where the wiki gets information not in the code.
- **Review model:** wiki updates land as a separate PR (or a separate commit on the same PR) so they're diffable and rejectable.
- **Failure mode to avoid:** wiki drifts and nobody trusts it. The only defenses are (a) automation that actually runs, (b) lint that catches drift, (c) keeping the wiki small enough that humans skim it.

### 7.2 Phase 3: Jira + Confluence

- **Auth:** OAuth + API tokens. Probably an `mcp__atlassian__*` MCP server is the cleanest route — already exists in some form. Don't reinvent.
- **What we pull:**
  - **Jira:** tickets linked to PRs (via branch name, commit trailers, smart commits). Each becomes a citation in `wiki/decisions.md` or a per-ticket page in `wiki/jira/`.
  - **Confluence:** specific spaces/pages flagged in config. Pulled into `wiki/confluence/` as raw + summary, same pattern as `pin-llm-wiki`'s `raw/` + `wiki/sources/`.
- **Privacy:** wiki may now contain ticket content that shouldn't leave the repo. Config flag for "redact internal-only fields," respected on ingest.
- **Linking model:** every wiki page can carry `Linked tickets:` and `Linked Confluence pages:` footers. Agents follow these when answering "why was this changed?".

### 7.3 Phase 4: Wiki-of-wikis

- Hardest phase. Real value: cross-service "where does this concept live?"
- **Approach A:** central org-wiki repo that pulls each repo's `wiki/` via submodule or scheduled sync, generates a meta-index. Cheap, works.
- **Approach B:** each repo's wiki publishes to a shared store (S3, GitHub Pages, Backstage), and a meta-tool federates. Heavier, better for non-engineers.
- **Cross-cutting concerns to handle:** shared glossary terms (same word, different meanings across repos), service catalog (who calls whom), ownership (CODEOWNERS aggregation).
- **Existing tool in this space:** [Backstage](https://backstage.io). We are not trying to be Backstage. We could *feed* Backstage. That might be a better framing.

## 8. Risks and things to watch

- **Stale wiki problem.** Solved only by Phase 2 working well. Phase 1 without Phase 2 has a 6-month half-life before people stop trusting it.
- **LLM cost on first-time generation of a large repo.** Need to estimate before committing to design.
- **Hallucinated architecture.** A confident wrong wiki is worse than no wiki. Citation discipline + lint must be enforced from Phase 1.
- **Skill name collision.** `pin-llm-wiki`'s slash command is `/pin-llm-wiki`. We'll want `/repo-llm-wiki` (or shorter, `/rlw`). Confirm before publishing.
- **Overlap with DeepWiki.** Honest answer: DeepWiki is better for "I want a hosted wiki for some random public repo." We are better for "I want a wiki that lives in my private repo, integrates with my Jira, and travels with my code." Different lanes.
- **Phase creep.** All four phases is ~a year of work. Phase 1 should ship in a couple of weeks and stand on its own.

## 9. What would prove or disprove the idea fast

A useful next step before any building:

1. Run **DeepWiki** against the target repo (or a clone if it's private). Read what it outputs. Decide whether what it gives us is enough — if yes, the answer might be "use DeepWiki, don't build."
2. Run **`pin-llm-wiki ingest`** pointing at the target repo's GitHub URL. See how close `pin-llm-wiki`'s GitHub source-type already gets to a Phase-1-ish wiki page. Possibly Phase 1 of `repo-llm-wiki` is just *deeper, multi-page* version of what `pin-llm-wiki` already produces for one external GitHub repo.
3. Manually write the Phase 1 wiki for the target repo — 6 pages, by hand, ~2 hours. Does it land? Do you actually use it? That tells us what the LLM should produce.

The third one is the most informative and the cheapest.

## 10. Sources

- Karpathy, [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- Cognition, [DeepWiki announcement](https://cognition.ai/blog/deepwiki) · [DeepWiki home](https://deepwiki.com/) · [Devin docs: DeepWiki](https://docs.devin.ai/work-with-devin/deepwiki) · [CognitionAI/deepwiki](https://github.com/CognitionAI/deepwiki)
- [AsyncFuncAI/deepwiki-open](https://github.com/AsyncFuncAI/deepwiki-open)
- [AIDotNet/OpenDeepWiki](https://github.com/AIDotNet/OpenDeepWiki)
- [sopaco/deepwiki-rs (Litho)](https://github.com/sopaco/deepwiki-rs)
- [context-labs/autodoc](https://github.com/context-labs/autodoc)
- [DocAgent paper, arXiv 2504.08725](https://arxiv.org/html/2504.08725v1)
- [divar-ir/ai-doc-gen](https://github.com/divar-ir/ai-doc-gen)
- [Building CodeWiki — Muhammad Raza](https://muhammadraza.me/2026/building-codewiki-compiling-codebases-into-living-wikis/)
- [Building a per-repo wiki with GitHub Actions — earezki.com](https://earezki.com/ai-news/2026-04-24-building-a-per-repo-wiki-that-actually-gets-read/)
- [AGENTS.md explainer — Builder.io](https://www.builder.io/blog/agents-md)
- [Atlassian: Confluence as a Knowledge Base](https://www.atlassian.com/software/confluence/resources/guides/extend-functionality/confluence-jsm) · [Atlassian AI in Confluence](https://www.atlassian.com/software/confluence/resources/guides/best-practices/atlassian-ai)
- [Docsie: Jira AI documentation 2026](https://www.docsie.io/blog/articles/jira-ai-documentation-integration-2026/)
- [Kodesage — AI docs for legacy code](https://kodesage.ai/blog/ai-documentation-tools-for-legacy-code)
- [Mintlify](https://mintlify.com/) (referenced via 2026 roundup)
- [Memgraph: GraphRAG for devs](https://memgraph.com/blog/graphrag-for-devs-coding-assistant)
- [GitNexus — client-side code knowledge graph](https://github.com/abhigyanpatwari/GitNexus)
- [Monorepos & AI — monorepo.tools](https://monorepo.tools/ai)
- [Augment Code: Monorepo vs Multi-repo AI](https://www.augmentcode.com/tools/monorepo-vs-multi-repo-ai-architecture-based-ai-tool-selection)
- [Backstage developer portal](https://backstage.io)
