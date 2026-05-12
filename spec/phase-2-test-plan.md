# repo-llm-wiki — Phase 2 test plan (`story`)

> **Pairs with:** `phase-2-prd.md`, `phase-2-implementation.md`  
> **Date:** 2026-05-12  
> **Author:** nenad  

This document describes **manual end-to-end testing** for Phase 2, using **`/repo-llm-wiki story`**, plus a **repeatable demo** for stakeholders (init baseline vs story delta).

---

## 1. Canonical fixture repo

| Item | Value |
|------|--------|
| Repo path | `/Users/nenaddjordjevic/GolandProjects/tfaws-dregdata-dhs-app-authorisation` (author's local fixture; substitute any go-monorepo with `cmd/` + `infra/*.tf`) |
| Starting state | `main` checked out, **clean** `git status`, **no** `wiki/` (no `wiki/index.md`) |
| Remote | `origin` present with `origin/main` (required for auto base-detection in PRD §5.2; if absent, scenarios fall back to `--since`) |

Verify before any run:

```bash
cd <fixture-repo>
git status                                    # clean
git branch --show-current                     # main (or override base via --since)
git branch -r | grep -E 'origin/(main|master|develop)$'   # auto-detect target exists
test ! -f wiki/index.md && echo "no wiki OK" || echo "FAIL: wiki exists"
```

---

## 2. Branch vs worktree for `init` (recommendation)

**Default (simplest for this test): use a feature branch in the same directory.**

1. `git checkout -b phase2-test/init-baseline main`
2. Run `/repo-llm-wiki init`
3. Commit the wiki when you are happy (optional for the *test*; PRD says the tool never commits — you may commit only to freeze a demo snapshot).

**Why not a worktree first?** A worktree is optional here. It helps when you must keep `main` checked out elsewhere or want two full IDE windows. For a linear demo (“clean main → branch → init → story”), one clone + feature branch is enough.

**When to use a worktree:** If you want **two folders** without switching — e.g. folder A frozen at “post-init commit”, folder B where you keep hacking — add a second worktree on another branch and copy or merge as needed. Not required for the baseline test below.

---

## 3. Demo narrative (for the team)

**Message:** Phase 1 **`init`** establishes the wiki from the repo; Phase 2 **`story`** updates **only the pages touched** by the dev’s change and adds a **skim-readable** line in `wiki/log.md`.

**Artifacts to show:**

| Artifact | What it proves |
|----------|----------------|
| `git diff` / folder diff for `wiki/` **after `init`** (vs no wiki) | Breadth of first-time generation |
| Same **after `story`** (vs post-init tree) | Narrow, intent-aware updates + new log entry |
| `wiki/log.md` | `init` / `refresh` entries vs new `## … — story` block (PRD §5.5) |

**Capture method (recommended):**

1. After `init`, optionally commit: `git add wiki AGENTS.md && git commit -m "wiki: init baseline (phase 2 test)"` — gives a stable ref for diffs.
2. After implementing the feature (before `story`), commit code: `git commit -m "feat: <story title>"`.
3. Run `/repo-llm-wiki story "<human description>"`.
4. Show:
   - `git diff <init-commit>..HEAD -- wiki/` — everything the wiki gained from init through story (cumulative), **or**
   - `git diff HEAD~1..HEAD -- wiki/` — **only** what `story` changed if you commit wiki separately.

For a **side-by-side file view** without intermediate commits, tar or copy `wiki/` to `wiki.after-init/` before feature work, then after `story` run:

```bash
diff -ruN wiki.after-init wiki | less
```

(Adjust paths if you snapshot outside the repo.)

---

## 4. Scenario A — Happy path (“new Lambda + DynamoDB read” stub)

Concrete handler names, queue names, and ticket id are **TBD**. Replace placeholders when the story is defined.

### A.1 Preconditions

- §1 verification passes.

### A.2 Baseline wiki (`init`)

1. `git checkout main && git pull` (if applicable)
2. `git checkout -b phase2-test/lambda-<story-id>`
3. `/repo-llm-wiki init`
4. **Assert:**
   - `wiki/index.md` exists.
   - `wiki/log.md` has an `## … — init` entry as the first block (Phase 1 init writes this; see `init.md` §5.4).
   - `/repo-llm-wiki lint` passes — this is the pre-story baseline; if it fails, fix before continuing or the story-side `lint` check is moot.
5. **Optional baseline snapshot:** duplicate tree for demo diff (§3).

### A.3 Simulated dev story (implementation TBD)

Goal: touch a **small, realistic** slice of the monorepo so **page-mapping** (PRD §5.3) routes to **`lambdas/`** (and **`infra/`** if TF changes).

Suggested touch set (adjust to repo layout):

- New or extended Lambda **handler** under `cmd/` or `app/cmd/` (as used by target repo).
- **Tests** beside or under same module.
- **Terraform** (or repo’s IaC layout): DynamoDB table or IAM/policy wire-up read by that Lambda.

Keep the diff **narrow** (< ~10 source files if possible) to stay under Phase 2 “typical story” expectations (PRD §7 success criteria).

### A.4 `story` invocation

Pick a description **≥ 10 characters** (PRD §5.6), e.g.:

```text
/repo-llm-wiki story "Add Lambda reading device state from DynamoDB for authorisation checks"
```

### A.5 Assert (post-story)

| # | Check | How to verify | Source |
|---|--------|---------------|--------|
| 1 | Console lists **affected wiki pages**; count ≤ 3 preferred | Read the `Wiki story logged (N pages updated)` completion block from `story.md` | PRD §4, §7 |
| 2 | Only listed pages changed; other `wiki/**/*.md` byte-identical to pre-story (except `log.md`) | `git diff --name-only -- wiki/` matches the reported list ∪ `wiki/log.md` | PRD §5.4 |
| 3 | `wiki/log.md` prepended entry has all required fields | Head of `wiki/log.md` contains `Description / SHA / Branch / Files changed / Pages updated / Summary / Tool` | PRD §5.5, lint W4 |
| 4 | `AUTO_SUMMARY` reads like factual "what"; description reads like "why" | Manual eyeball | PRD §5.5 |
| 5 | No relative-path source citations introduced (`../app/...` etc.) | `lint` E2 — covered by check 6 | PRD §5.4, lint E2 |
| 6 | `/repo-llm-wiki lint` passes | Run lint, expect `Result: PASS` | PRD §7 |
| 7 | No unauthorised top-level category page invented | `lint` E8 — covered by check 6 | `init.md` §3.1 (closed set) |

### A.6 Unmatched files (spot-check)

If the tool prints **unmatched** paths (impl plan §2 step 4, §9):

- Confirm they are benign or intentional; if critical paths are unmatched, cite gap for Phase 2.1 / citation fixes.

---

## 5. Scenario B — Guards and edge cases

Each row maps to a guard or edge case in `story.md`. Run **after** a healthy wiki exists (complete A.2 first; reuse a throwaway branch where convenient).

| ID | Steps | Expected | Source |
|----|--------|----------|--------|
| B.1 | `story ""` (empty) | Print usage block; exit | story.md Guard C |
| B.2 | `story "short"` (< 10 chars) | Print usage block; exit | story.md Guard C |
| B.3 | On `main` with clean tree; `FORK_SHA == HEAD_SHA` | Stop: *"No changes to summarise. Nothing to update."* | story.md Step 3a |
| B.4 | Edit **only** `wiki/foo.md`; no code changes | Stop: *"Story touches only `wiki/` — nothing to summarise."* | story.md Step 3a edge case |
| B.5 | `--since <ref>` happy path: branch with 2+ commits, run `story "..." --since main` | Diff matches `git merge-base HEAD main..HEAD` + working tree; pages update; log entry present | story.md Step 2a |
| B.6 | Remote has neither `main`, `master`, nor `develop` (rename a branch in a scratch clone, or `git remote remove origin`) | Stop with auto-detect failure message; instructs use of `--since` | story.md Step 2a |
| B.7 | Shallow clone (`git clone --depth=1 ...`) | `merge-base` returns nothing; stop with hint to `git fetch` or pass `--since` | story.md Step 2b |
| B.8 | Detached HEAD (`git checkout <sha>`) with at least one uncommitted change | Branch field in completion + log entry reads `(detached at <sha[0:7]>)`; otherwise proceeds | story.md edge case table |
| B.9 | Description contains a pipe (`"adds \| splitter"`) or line-start `##` | Log entry remains a well-formed Markdown bullet (pipe escaped to `\|`, `##` to `\#\#`) | story.md edge case table |
| B.10 | Branch adds a brand-new Lambda dir under `cmd/foo/` not referenced by any wiki page | Story routes its files to `wiki/lambdas.md` or flags them unmatched; completion summary tells dev to run `refresh` for new top-level entries | story.md Step 4 / edge case |
| B.11 | Synthetic large change: touch > 100 files (`for f in $(...) ; do echo "// noop" >> $f; done` on a throwaway branch) | Tool prints threshold warning and asks for confirmation before continuing; suggests `refresh` | story.md Step 3c |
| B.12 | Synthetic broad change: touch files cited by 7+ wiki pages | Tool prints `AFFECTED_PAGES > 6` warning before fanning out | story.md Step 4 |

---

## 6. Scenario C — Cross-cutting vs single-module

| ID | Goal | Outline |
|----|------|---------|
| C.1 | Single-module | One Lambda handler + tests only → expect **1–2** pages |
| C.2 | Cross-cutting | TF + Lambda + shared package → multiple categories; verify list matches touched paths and stays within cap behaviour |

(Reuses impl §8 items 1–2 with explicit pass/fail for the target repo.)

---

## 7. Definition of done (Phase 2 from a testing perspective)

- [ ] Scenario **A** passes on the fixture repo from **clean main → init → synthetic story → story**.
- [ ] Demo package prepared: diff or commits showing **init** vs **post-story** wiki (§3).
- [ ] Guards **B.1–B.4** (input + no-op guards) behave as specified.
- [ ] Branch-base detection **B.5–B.7** covered: `--since` happy path, auto-detect failure, shallow clone.
- [ ] At least one of **B.10–B.12** (unmatched / large / broad) verified to confirm thresholds and unmatched-routing fire.
- [ ] **`lint`** passes after **`story`** on the happy path (E1–E9 clean, W4/W5 also clean).
- [ ] Deviations from PRD §5.3 / impl §6 (e.g. fallback page names) are noted in test notes / research follow-up.

---

## 8. Optional checklist (session log)

Copy for each run:

```text
Date:
Branch:
Git clean before init: Y/N
init: Y/N — lint: 
Story description:
Files changed (count): 
Pages reported updated: 
Unmatched files: 
lint after story: 
Notes:
```

---

## 9. References

- Phase 2 PRD: `spec/phase-2-prd.md`
- Phase 2 implementation: `spec/phase-2-implementation.md`
- Phase 1 skill: `skills/repo-llm-wiki/init.md`, `refresh.md`, `lint.md`
