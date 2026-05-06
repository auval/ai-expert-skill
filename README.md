# ai-expert — Claude Code Plugin

Let AI agents become true experts on your repo. Save tokens by maintaining a
lightweight `expert/` folder of non-obvious domain facts, so agents never
re-explore the same ground twice.

## How it works

Every non-trivial software project has domain knowledge that is expensive for an AI
agent to reconstruct from code alone — things that require tracing multiple files,
understanding historical decisions, or knowing which file is the single source of truth.

This plugin teaches Claude to:
- Read `expert/index.md` at the start of every task (small fixed cost, large savings).
- Selectively read one relevant domain file when the task touches that domain.
- Update domain files at task end via a retrospective protocol.
- Initialize the `expert/` folder from scratch if it doesn't exist yet.

## Installation

**Step 1 — Add the marketplace** (once per machine):
```
/plugin marketplace add auval/ai-expert-skill
```

**Step 2 — Install the skill:**
```
/plugin install ai-expert@auval-plugins
```

## Project structure after setup

```
your-project/
└── expert/
    ├── index.md        ← always read first; index of all domain files
    ├── auth.md         ← example domain file
    ├── billing.md      ← example domain file
    └── data-model.md   ← example domain file
```

## What goes in an expert file

Only facts that are all three of:
- **Important** — affect correctness, architecture, or common agent mistakes.
- **Non-obvious** — cannot be inferred from a single file read or grep.
- **Stable** — not task history, not changelogs, not bug diaries.

## What does NOT go in an expert file

- Code obvious from one read.
- General architecture overviews.
- Task history or bug diaries (unless they reveal a stable domain invariant).

## License

Apache 2.0
