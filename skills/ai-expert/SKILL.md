---
name: ai-expert
description: >
  This skill activates at the start of every non-trivial task in any software project.
  It manages an expert/ folder that stores domain knowledge that is important, non-obvious,
  and expensive for future AI agents to reconstruct from code alone.
  Use when the user starts a new task, mentions "expert files", "domain knowledge",
  or asks you to remember non-obvious project facts across sessions.
version: 1.0.0
---

# AI Expert — Domain Knowledge Manager

This skill maintains an `expert/` folder in the project root that future agents read
to avoid re-exploring non-obvious domain knowledge.

## On Every Non-Trivial Task: Startup Protocol

1. Check whether `expert/index.md` exists in the project root.
2. If it exists, **read it** before doing anything else. Treat this as a fixed, small context
   fee that saves far more tokens later.
3. After reading the index, read **only the one relevant domain file** whose topic clearly
   overlaps with the current task — do not read the whole folder.
4. If `expert/index.md` does not exist, offer to initialize it (see Initialization below).

## When NOT to Read Domain Files

- Narrow, local edits whose ownership is obvious from nearby code.
- Tasks where no domain file topic overlaps with the work.
- Keep context as small as possible — the index is always worth reading; individual files are selective.

## What Expert Files Must Contain

Each domain file (`expert/<domain>.md`) must contain only:
- High-level architecture of the domain.
- The files that actually own the domain, and what each file contributes.
- Non-obvious, high-value deductions that usually require tracing multiple files, flows,
  or semantic layers.

**Do not include:**
- Direct restatements of code that are obvious from a quick read.
- Broad project documentation or architecture overviews.
- Task history, bug diaries, or changelog notes — unless they reveal a stable domain invariant.

## On Every Non-Trivial Task: Retrospective Protocol

At the end of each task, ask:
> "What did I need to explore in this session that a future agent shouldn't have to?"

If the answer is something **important AND non-obvious** that cannot be inferred from a single
grep or standard framework conventions, update the relevant expert file (or create a new one).

**Rules for retrospective updates:**
- Only add insights useful to future agents working on the **same domain but a different task**.
- Do NOT add insights that are too narrow or task-specific.
- Remove or rewrite obsolete facts.
- Keep all entries concise and high-signal.

## Initialization: Creating expert/ From Scratch

When `expert/index.md` does not exist, create it with this template:

```markdown
# Domain Experts

Purpose: save tokens for future agents by documenting domain knowledge that is important, non-obvious, and expensive to reconstruct from code alone.

Every agent that reads or benefits from this folder must keep it up to date:
- remove or rewrite obsolete facts
- add new useful domain facts when they are stable and non-obvious
- keep entries concise and high-signal

Boundaries:
- include only important facts that usually require tracing multiple files, flows, or semantic layers
- do not restate code that is obvious from one file read or one grep
- do not turn this folder into general documentation, task history, or changelog notes

When to read:
- read this index at the start of every task
- treat the index as the fixed context fee; individual expert files remain selective
- read a domain file only when the task clearly touches that domain and the file is likely to reduce exploration
- prefer reading one relevant domain file, not the whole folder

When not to read:
- do not read domain files by default
- do not read them for narrow local edits whose ownership is already obvious from nearby code
- keep context as small as possible

Each expert file should contain only:
- high-level architecture of the domain
- the files that actually own the domain, and what each file contributes
- non-obvious, high-value deductions that usually require tracing multiple flows

Do not include:
- direct restatements of code that are obvious from a quick read
- broad project documentation
- task history or bug diary details unless they reveal a stable domain invariant

Retrospective:
- what did you need to explore in this session that a future agent shouldn't have to?
  if you found something important AND non-obvious that can't be inferred from a single grep or standard Android conventions,
  then update the relevant the expert file for the domain(s), or create a new expert file if not existing.
- DO NOT add too narrow insights! only insights useful for future agents working on the same domain but A DIFFERENT TASK.

## Entries
(none yet — add domain files as you discover non-obvious knowledge)
```

Then add domain files as you discover non-obvious knowledge during tasks.

## expert/index.md Maintenance

The index must stay in sync with the actual files in `expert/`:
- Add a link whenever a new domain file is created.
- Remove the link when a domain file is deleted.
- Keep each entry as a single line: `- [Domain Name](./filename.md)`
