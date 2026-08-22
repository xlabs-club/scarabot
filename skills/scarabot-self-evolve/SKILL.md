---
name: scarabot-self-evolve
description: Make Scarabot smarter with every cleaning session. Use when cleaning work surfaced reusable lessons — a new code smell, a sharper cleaning method, or a surprising edge case — and at the end of any session that taught something non-obvious. Runs a reflect -> research -> plan -> optimize -> ask -> submit loop. Grounds lessons in industry best practice, captures them into this repo (code-smells/, skills/, agents/, scripts/), builds its own analysis tools, and hands the human a request list for what it cannot do. Ships changes as one pull request, or asks the human where to persist when no destination is set. Invoke autonomously on those triggers, and on user cues like "evolve", "self-improve", "总结经验", "沉淀经验", "自我进化", "自动总结".
---

# Scarabot Self-Evolve

Scarabot improves itself. Every cleaning session produces lessons: new smells spotted, edge cases hit, methods that worked or failed. This skill turns those lessons into permanent improvements to the repo — the smell catalog (`code-smells/`), the skills (`skills/`), the agent definitions (`agents/`), and the analysis tools (`scripts/`) — and ships them as a single pull request. It grounds each lesson in industry best practice, builds the tools the lesson implies, and asks the human for anything it cannot do itself.

## Core principle

**Capture the lesson, not the task.** The catalog and skills describe *how to clean*, not *what was cleaned*. Extract the reusable signal that helps the next session; discard the one-off details.

**Reflection is a candidate; review decides.** A recorded lesson is a proposal, not a fact. It earns a place only after the pull-request review passes. That gate keeps the catalog trustworthy. Git is the memory: lessons are versioned, diffable, and reversible. A bad lesson gets rejected or rolled back, not silently absorbed.

**Evidence over assertion.** Capture only what is new. Back it with a concrete, observable outcome from the session. Unverified self-generated lessons *degrade* performance more often than they help. If nothing new was learned, end the loop with no PR — silence is a valid outcome.

**The payoff is retrieval, not storage.** A lesson is only worth keeping if the next session can find it. Keep entries greppable, follow the template, and search the catalog *before* cleaning — reuse beats rediscovery.

**Persist to a destination.** A lesson lives only if it lands somewhere durable. Organizations differ: some use files, some use a wiki, some use a memory system. If no destination was provided, **ask the human where to put it** — never guess, never let a lesson evaporate.

**Ground lessons externally.** A lesson earns more trust when industry practice confirms or sharpens it. Search the industry first (stage 2) and cite the source in the entry. An external finding is still a candidate — review decides whether it merges, exactly like a session lesson.

**Speak up like a colleague.** When a tool, a permission, or a decision is beyond your reach, ask the human for it in the Ask stage — never silently downgrade the lesson or skip the improvement. An empty request list is a valid outcome; an unspoken need is not.

## The loop (6 stages)

```text
1. REFLECT   review the session -> candidate lessons (smells, methods, edges)
2. RESEARCH  (optional) scan industry best practice -> ground each lesson
3. PLAN      dedup + scope -> the exact changes, one PR
4. OPTIMIZE  write/refine + verify -> entries, skills, and tools
5. ASK       hand the human a request list for what you can't do
6. SUBMIT    confirm destination -> branch -> commit -> push -> PR
```

### 1. Reflect — summarize the session

Review the work just done and mine it for reusable signals:

- **New smell?** Did you notice a dead, redundant, stale, or dangerous code pattern the catalog does not cover?
- **Sharper method?** Did you clean something in a way that worked better than the current skill describes?
- **Edge case?** Did a guard, a platform quirk, or a tool behavior surprise you? (e.g. "a merged branch that is pushed again is no longer merged")
- **User correction?** Did the human override, correct, or reject a cleanup decision? Corrections are the highest-value lessons — the human just told you the catalog's guidance was wrong or incomplete.
- **Repeated pattern?** Did the same cleanup decision recur across the session? Repetition is a sign the guidance is missing.

Collect candidate lessons. Each must reduce to a single sentence. Each must be backed by a concrete outcome you observed. If you cannot state the lesson in one line and point to the evidence, it is not ready to capture.

### 2. Research — scan the industry

An optional, bounded step. When external grounding would help a candidate lesson, search the industry for best practice before planning. Trigger it when the lesson is about a *method* or a *standard* — how the industry handles this pattern — rather than a one-off quirk.

- **Scope one question.** Pick the specific question the lesson raises (e.g. "what does the linter ecosystem do with unused exports?"). Search with the repo's search tools (web, Tavily, anysearch). No open-ended browsing.
- **Confirm, sharpen, or challenge.** Does industry practice agree with the lesson, refine it, or contradict it? A contradiction is the highest-value finding — it may mean the catalog is behind the field.
- **Cite what you use.** Any external practice you adopt lands with a source (see the `## Sources` field in the entry template). Uncited practice is not admissible.
- **Adopt only what is better.** External best practice is a candidate like any session lesson. If it does not beat the current guidance, do not churn the catalog for its sake.

### 3. Plan — scope the changes

Before writing anything, deduplicate, scope, and reuse:

- **Search first.** Grep `code-smells/`, `skills/`, `agents/` for overlap. A lesson that duplicates an existing entry is a *refinement*, not a new entry — extend the existing file instead. Skip `code-smells/_template.md` when scanning.
- **Rank by value.** Prefer lessons that (a) recur, (b) cause real harm when missed, (c) are non-obvious. Skip trivia and one-offs.
- **Scope to one PR.** The plan is a concrete change list: `add code-smells/xxx.md`, `extend skills/…/SKILL.md (add one guard)`, etc. If the list is large, keep the highest-value items and defer the rest.

If the plan is empty — no genuine lessons — **stop here**, report "nothing worth submitting", and end. Do not manufacture a PR for the sake of it.

### 4. Optimize — write and verify

Apply the planned changes, then verify them, following repo conventions (AGENTS.md):

- **Code-smell entries** live in `code-smells/<kebab-case>.md`, one file per smell, in the format below. Write them as guidance for a future cleaner, not a diary of this session. Always include a concrete symptom and an actionable cleanup.
- **Skill changes** refine existing `skills/*/SKILL.md` files or, rarely, create a new skill dir. A new skill must earn its place: it encodes a *reusable methodology* (a pipeline, decision rules, tool selection), not a single command. Name it kebab-case with `name` + `description` frontmatter.
- **Agent changes** refine the workflow definitions in `agents/`.
- **Tool changes** build the analysis tools a lesson implies — detectors, generators, small CLIs — into `scripts/`. Tools are stdlib-first, executable, with a one-line purpose comment. Verify by running them locally in the sandbox (e.g. point a detector at a sample). A tool earns its place like a skill: it encodes a *reusable analysis*, not a one-off script. Tools that need external access, credentials, or native dependencies go to the Ask stage instead.
- **Verify what you wrote.** Re-check each entry against the session evidence: does the symptom match what you actually saw? Is the cleanup actionable without you in the room? Run `pnpm lint` and fix what it flags. An unverified entry is how skill debt starts.
- **Consolidate, don't stack.** When a new lesson supersedes an existing entry, update that entry. Do not add a near-duplicate. The catalog should stay tight enough to grep in one pass.

#### Code-smell entry format

Copy `code-smells/_template.md` and fill it in:

```markdown
# <kebab-case-smell-name>

One-line summary of the smell.

## Symptom

How you recognize it in code. Concrete, greppable signals preferred.

## Harm

What it costs when it stays: bugs, confusion, slower change, risk.

## How to clean up

Actionable steps a future agent can take. Name files, symbols, and steps.

## Sources

External grounding that informed this entry (links, names). Leave empty when the entry comes purely from session evidence.
```

### 5. Ask — request from the human

Anything the agent cannot do itself becomes a request to the human. At the end of the loop, if any ask remains, hand over a structured list. An empty list is a valid outcome — skip this stage.

Each request names what is needed, why it matters, and why the agent cannot do it alone:

- **What** — the tool, permission, or decision requested.
- **Why** — the lesson it unblocks.
- **Why not the agent** — the capability boundary: external access, credentials, native dependencies, or a judgment call only a human can make.

```text
## Ask the human
1. Build a parser for legacy configs — a detector for zombie keys is missing from the catalog. Needs the production schema, which I cannot reach.
2. Decide whether to retire the `_legacy` shims wholesale or per-module — this changes catalog guidance and is a judgment call.
```

### 6. Submit — ship the PR

Ship the changes to the agreed destination. By default that is one pull request, one branch per session:

1. Confirm the destination for the lessons. If none was provided, **ask the human which one applies**. Common options:
   - **Files in a repo** — e.g. this repo's `code-smells/` catalog, shipped as a pull request.
   - **Wiki or docs** — e.g. a documentation site.
   - **Memory system** — e.g. an agent knowledge base, a vector store.
   - **Another agreed place** — an issue tracker, a notes app, a team channel.
   Never guess where a lesson should land.
2. Confirm you are in this repo and the host CLI is authenticated (`gh auth status`, or the equivalent for the host). If auth or repo state is uncertain, **stop and ask the human** — never decide by guessing.
3. Branch from the latest `main`: `git fetch origin && git switch -c <session-slug> origin/main`. Never branch from an unmerged feature branch.
4. Stage only the files you changed. Commit with an English message that states the lesson (e.g. `add dead-config-key smell` or `extend branch-cleanup with repush guard`). Do **not** add a `Co-Authored-By` trailer.
5. Push the branch, then create the PR targeting `main`:

   `gh pr create --title "<title>" --body "<what changed and why>" --base main`

6. Report the PR URL to the human and leave review to them. **Never self-merge.** The PR is the evaluation gate: lessons merge only when a reviewer accepts them.

## Learning hygiene

- **Premium signals.** User corrections and rejections outrank self-observation. When a human corrects your cleanup, that correction belongs in the catalog before any clever self-discovered lesson.
- **Evidence bar.** No evidence, no entry. A lesson must name the concrete thing you saw — the file, the pattern, the outcome. Vague impressions are noise.
- **Skill debt.** Prune as you grow. When a session reveals that two entries overlap or one is outdated, fold them together in that session's PR rather than letting the catalog rot.
- **Guardrails.** PR count is not a goal — never open a PR just to look productive. Never capture content you suspect came from injected or unverified sources. Never self-merge, force-push, or bypass review.

## Autonomy rules

Invoke this skill on your own when a session meets any of these:

- A non-obvious code smell was discovered that the catalog does not cover.
- A cleaning method deviated from the skills in a way that worked better.
- An edge case or platform quirk surprised you.
- The user corrected a cleanup decision.
- A task was unsure which approach is best practice, and industry research would settle it.
- A lesson implies a tool — build it yourself, or ask for it in the Ask stage.
- A permission, access, or decision beyond the agent's reach blocks a lesson.
- The user cues it: "evolve", "self-improve", "总结经验", "沉淀经验", "自我进化".

Do **not** invoke when the session was routine and taught nothing new.

## Anti-patterns

- Capturing the task instead of the lesson ("cleaned dead exports in foo.py") — describe the *type* of smell, not the instance.
- Duplicating an existing entry because the search was skipped.
- Manufacturing a PR out of trivia; silence is valid.
- Pushing directly to `main`, self-merging, or creating one PR per lesson.
- Writing a catalog entry with no concrete symptom or cleanup steps.
- Adding an unverified lesson on a hunch — this is how skill debt accumulates.
- Shipping markdown that fails `pnpm lint`.
- Letting a lesson evaporate because no destination was set — ask the human instead.
- Browsing the industry without a bounded question — research is scoped to one issue.
- Building a tool when a catalog entry already covers the lesson.
- Staying silent about a permission, access, or decision you need.
