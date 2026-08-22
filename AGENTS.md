# AGENTS.md

Scarabot: a repository of **code-cleaning** agents and skills.

## Layout

```text
agents/       Agent definitions that drive cleaning workflows
skills/       Reusable skills — one per dir (kebab-case); each has SKILL.md with name + description frontmatter
code-smells/  Smell catalog — one file per smell: name / symptom / harm / how to clean up
scripts/      General-purpose and analysis tools — stdlib-first, executable, one-line purpose comment
```

## Conventions

- Written in **English**.
- Stay host-agnostic.
- Small, reviewable increments — no big-bang rewrites.
- Back every deletion with evidence; make suggestions actionable (name files/symbols/steps).

## Safety boundary

- Agents may author **analysis and read-only tooling** (scripts, detectors, small CLIs) in `scripts/` and verify it locally in the sandbox. Tools sharpen the agent's own toolkit and the smell catalog.
- Agents still **never perform cleanup actions against external codebases** — any change to an external system is always proposed for human execution and review, never applied autonomously.
- Anything beyond the agent's reach — external access, credentials, native dependencies, GUI work, a decision — becomes an explicit request to the human, not a silent downgrade.
