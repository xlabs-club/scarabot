---
name: safe-branch-cleanup
description: Safe git branch cleanup that removes dead/stale branches without ever deleting useful ones. Use when the user wants to clean up, prune, or delete stale branches — locally or on a git host (trigger phrases: "branch cleanup", "prune branches", "delete stale branches"). Teaches the safety criteria and the audit -> backup -> review -> delete -> verify pipeline; prefers the host CLI (`git`/`gh`, or the equivalent), falls back to the REST API, and guides writing small ad-hoc scripts when neither fits.
---

# Safe Branch Cleanup

Keep repositories tidy by deleting branches that are **provably dead** — and never delete one that is still in use. This skill is a **methodology**: it teaches the decision rules, the pipeline, and which tools to reach for. It ships no programs; write small ad-hoc scripts when no CLI covers your platform.

## Core principle

Deleting a branch is cheap to get wrong and feels irreversible. The whole workflow exists to make one guarantee: **a branch is only deleted when every independent safety signal agrees it is dead.** Auditing is read-only; deletion is a separate, explicit, verified step.

When any signal is uncertain or can't be verified, **stop and ask the human to confirm before executing** — never decide by guessing.

A failed or inconsistent API/CLI call is itself a form of uncertainty: if a check errors, returns partial/conflicting data, or yields a state you can't fully verify, treat the branch as **not deletable** — flag it for manual review and move on.

A branch is safe to delete only when **all** of these hold:

| Signal | Meaning |
| --- | --- |
| Merged | Tip is an ancestor of the default/trunk branch |
| Not protected | Not a protected branch on the git host |
| No open PR | No open pull request uses it as source |
| Cooldown | Last commit is older than N days (default 30) |
| Name is not special | Never `main`/`master`/`develop`, `release/*`, `hotfix/*`, or team-protected prefixes |

The name exclusions and the protected check are **hard guards**: they apply even if the audit data says the branch is merged.

Unmerged branches are **never auto-deleted** — no matter how old or stale. If one should go, that's a per-branch human decision, reviewed one at a time.

Safety is the first priority; never trade it for tidiness or speed.

## Before you start (preflight)

- Confirm the tool's auth works (`gh auth status`, or the equivalent for the host).
- Confirm the repo scope — audit only the intended repos.
- Confirm cooldown days and exclusion patterns for this run.
- Confirm the delete target is the right remote/environment.

## The pipeline (5 stages)

```text
1. AUDIT    read-only scan -> candidate list + full report + error list
2. BACKUP   capture repo + branch + tip_sha for every candidate
3. REVIEW   human confirms the list
4. DELETE   dry-run preview -> explicit flag -> delete, resumable
5. VERIFY   deleted branches gone, default branch intact
```

### 1. Audit (read-only)

Enumerate branches; for each, compute the five signals. Never mutate anything. Produce:

- a **candidate list** (all signals hold),
- a **full report** (every branch and why it did/didn't qualify — this is what a human reviews),
- an **error list** (repos that couldn't be reached — exclude them, don't guess).

Two audit-time details matter:

- **Live merged check.** "Merged" must be computed at audit time, not trusted from cached state. A branch that was merged and then pushed again is no longer merged — recompute, don't cache.
- **Rate limits.** Auditing many branches hits the API hard. Sleep between calls, retry on 429/5xx with backoff, and cache raw responses so re-runs are cheap.

### 2. Backup manifest

Before deleting anything, write a manifest to a **durable, searchable file** in a `backup/` dir outside the cleaned repos — one row per candidate: `repo, refname, tip_sha` + author/date. Both identifiers must be **complete**, because the manifest must restore the branch exactly:

- `refname` is the **full branch name** (`refs/heads/<branch>`), never the short form — so the exact ref is recreated on restore.
- `tip_sha` is the **full 40-char commit id**, never a short SHA — short IDs can be ambiguous across the repo's history.

The manifest is metadata pointing at server state; restore only works while the server still holds the objects. Backup depth depends on GC exposure:

| Branch kind | Object reachability | Server GC threat | Backup needed |
| --- | --- | --- | --- |
| Merged (auto) | Reachable via default branch | Low — only if default's history is rewritten later | Manifest alone |
| Unmerged (human-approved) | Unreachable | **High — host `git gc` prunes within days** | `git bundle` mandatory |

For every unmerged branch (or when in doubt), capture the full object set locally:

```bash
git bundle create backup/<branch>.bundle refs/heads/<branch>
git bundle verify backup/<branch>.bundle
```

Restore:

```bash
git push origin <tip_sha>:refs/heads/<branch>   # merged: object still on server
git fetch backup/<branch>.bundle 'refs/heads/<branch>:refs/heads/<branch>'
git push origin refs/heads/<branch>             # unmerged: recover from bundle
```

**Verify the restore** — `git rev-parse refs/heads/<branch>` must equal the recorded `tip_sha`, then drop any temporary local ref. "Restored" only counts if the ref points at the recorded commit.

### 3. Human review + confirm

Present the audit report — totals, per-repo distribution, the oldest-sample rows, and unreachable repos — and ask for explicit confirmation. Generate the to-delete list *only after* the human confirms. Review is where mistakes get caught before they cost anything.

### 4. Dry-run, then delete

The delete step re-validates every row against hard-coded guards the input list cannot bypass:

- never delete the default branch,
- never delete protected branches,
- never delete names matching the exclusion patterns,
- **current tip still equals the recorded `tip_sha`** (compare-and-swap): if the branch advanced since backup, abort that row and re-backup — the manifest must describe exactly what gets deleted.

If any row violates a hard guard, **refuse the whole run** — do not skip-and-continue.

- Dry-run is the default: print what *would* be deleted, do nothing.
- **Canary batch first**: delete a small sample (3–5 branches) and verify before the full run — catches systematic errors (bad guard, wrong scope) cheaply.
- **Pace the deletes**: sleep between delete calls (at least ~1s), keep concurrency at 1, and retry 429/5xx with backoff — deletes are writes on the git server, so rate-limit even harder than the audit to avoid overwhelming it.
- An explicit flag (`--yes`) turns on actual deletion.
- Log every outcome (success/failure) so the run is **resumable** — on re-run, skip rows already recorded as deleted.

### 5. Verify

Re-fetch every deleted branch: it should be gone (404 / not found). Then confirm each repo's default branch still exists. A small post-delete check is what turns "I think it worked" into "it worked".

## Tool selection

Reach for the platform's CLI first; fall back to its REST API; write a small ad-hoc program only when neither fits.

### Local git

- Branches merged into default: `git branch --merged <default>`
- Is a branch tip an ancestor of default: `git merge-base --is-ancestor <branch> <default>`
- Last-commit dates for all branches: `git for-each-ref --format='%(refname:short) %(objectname) %(committerdate:iso8601)' refs/heads/`
- Delete locally: `git branch -d <branch>` — **refuses** to delete unmerged branches by design; that built-in guard is your friend. `-D` only after explicit review.
- Prune stale remote-tracking refs: `git remote prune origin`
- Delete a remote branch: `git push origin --delete <branch>`

### Host CLI (`gh` as the example)

Reach for the host's CLI. Commands below use `gh`; other hosts have an equivalent — same checks, different paths.

- List branches: `gh api repos/:owner/:repo/branches?per_page=100` — includes `protected` but **not** merged status.
- Merged check: compare against default — `gh api repos/:owner/:repo/compare/<default>...<branch>`; the branch is merged when `status` is `behind` or `identical` (base already contains all of the branch's commits). Recompute at audit time: a merged-then-repushed branch is no longer merged.
- Open PRs: `gh api repos/:owner/:repo/pulls?state=open` → read `head.ref`.
- Delete: `gh api --method DELETE repos/:owner/:repo/git/refs/heads/:branch` (the API refuses protected and default branches on its own — a natural guard).

### Writing an ad-hoc program (no CLI available)

When you must script it yourself, structure it as the pipeline above:

- **Separate read-only audit from destructive delete.** Two programs, or two explicit modes. The audit must never contain a delete path.
- **Cache raw API responses** to disk so re-runs are cheap.
- **Hard-code the guards** in the delete step: re-validate every input row against default-branch / protected / name-exclusion rules, and refuse the whole run on any violation.
- **Dry-run is the default**; deletion requires an explicit flag.
- **Rate-limit and retry**: sleep between calls, retry 429/5xx with backoff.
- **Resumable**: append every outcome to a log; on re-run, skip already-done rows.
- **Verify afterwards**: re-fetch deleted branches (expect gone) and confirm the default branch still exists.

## Adjustable knobs

- **Cooldown days**: how stale a branch must be before it is eligible. 30 is the default; tune it per repo.
- **Exclusion patterns**: trunk names (`main`/`master`/`develop`), `release/*`, `hotfix/*`, and any team-protected prefixes. Anything your release automation references must be excluded.
- **Repo scope**: audit a single repo or a fleet. Multi-repo works best from a plain list of repo URLs.

## Ongoing hygiene

The audit is read-only and cheap to re-run on cache — schedule it (cron/CI) to keep a fresh report of candidates. Deletion is **never** automated; every delete stays human-confirmed.

## Anti-patterns

- Deleting a branch because it *looks* old. Age is only one signal; an old, unmerged branch may be someone's long-running work.
- Trusting a snapshot of merged state. Recompute merged at audit time.
- Skipping the backup manifest because deletion is "reversible". The manifest is what makes recovery a 5-second command instead of archaeology.
- Deleting an unmerged branch without a `git bundle`. The manifest alone cannot restore it once the server's GC prunes the objects.
- Deleting in the same script that audits. Read-only and destructive must be separable.
- Assuming "it worked" without a verify pass.
