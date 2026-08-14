<div align="center">

# 🪲 Scarabot

**Rolling your legacy code out of the shadows, commit by commit.**

[English](./README.md) · [简体中文](./README.zh-CN.md)

</div>

---

## What is Scarabot?

Scarabot is an agent and skill repository dedicated to one mission:
**code cleanup** — taming the codebases everyone else is afraid to touch.

Legacy modules tangled beyond reason. Half-dead branches nobody dares delete.
"Temporary" shims that outlived three rewrites. Dead code masquerading as
features. Scarabot rolls through it all — identifies what's dead, what's
redundant, what's dangerous — and clears it away with surgical precision.

## What it does

- 🧹 **Dead code removal** — flags unused exports, unreachable branches, and
  imports that lead nowhere.
- 🔁 **Redundancy elimination** — detects duplicate logic and folds it into a
  single source of truth.
- 🗑️ **Stale cleanup** — hunts down deprecated paths, zombie configs, and
  commented-out history that belongs in git, not in the file.
- ⚔️ **Refactor with confidence** — proposes structural improvements and
  applies them in reviewable, bite-sized steps.

## Repository layout

```
scarabot/
├── agents/        # Agent definitions powering the cleanup workflows
├── skills/        # Reusable skills for specific cleanup tasks
├── code-clinic/   # Drop-in code for diagnostic cleanup sessions
└── README.md
```

## License

See [LICENSE](./LICENSE).