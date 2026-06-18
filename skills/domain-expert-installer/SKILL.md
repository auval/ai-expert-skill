---
name: domain-expert-installer
description: >
  One-time installer for the domain-expert system. Run once inside a target repo to create
  expert/index.md and wire Claude Code hooks (SessionStart, PreToolUse, PostCompact, PostToolUse)
  plus a git post-commit hook, so every future agent automatically reads accumulated domain
  knowledge at session start and is prompted to contribute back before context degrades — all
  enforced structurally by hooks, not soft directives.
  Use when the user asks to "install the domain expert", "set up expert/ files", "wire the expert
  hooks", or otherwise bootstrap cross-agent domain memory in a repository.
version: 1.0.0
---

# Domain Expert — Installer

This is a **one-time installation skill**. Its only job is to set up the domain-expert
instrumentation inside a target repo:

1. Create `expert/index.md` at the repo root.
2. Wire five Claude Code hooks + one git hook so every future agent reads `expert/index.md`
   automatically and is prompted to contribute back at the right moment.

Do not use this skill for runtime behavior. Once installation is complete, this skill's job is
done — the hooks take over.

---

## Step 1 — Identify the target repo

Confirm the path to the target repo root. This is the repo where `expert/index.md` will live and
where the hooks will be installed. If it is the current working directory, proceed. If not, ask the
user to confirm the path before touching any files.

All paths below are relative to this target repo root (written as `<target-repo>`).

---

## Step 2 — Create expert/index.md

Create `expert/index.md` inside the **target repo root** using the template in the
[Template](#template-expertindexmd) section below.

- If `expert/` does not exist, create the folder first.
- If `expert/index.md` already exists, do **not** overwrite it — tell the user it is already present
  and skip this step.

---

## Step 3 — Wire the hooks

Create the `.claude/hooks/` directory inside the target repo if it does not exist, then create the
five scripts below.

### Hook 1 — session-start-expert.sh (SessionStart)

Injects `expert/index.md` as a system-reminder at session start.

`<target-repo>/.claude/hooks/session-start-expert.sh`:

```bash
#!/usr/bin/env bash
# Injects expert/index.md as a system-reminder at session start.
set -euo pipefail
REPO_ROOT="$(git -C "$(dirname "$0")" rev-parse --show-toplevel 2>/dev/null || echo "")"
if [[ -z "$REPO_ROOT" ]]; then exit 0; fi
EXPERT_MD="$REPO_ROOT/expert/index.md"
if [[ -r "$EXPERT_MD" ]]; then
  echo "## expert/index.md (mandatory reading — domain knowledge for this repo)"
  cat "$EXPERT_MD"
fi
```

### Hook 2 — expert-gate.sh (PreToolUse: Write|Edit|MultiEdit)

Intercepts writes to any `expert/*.md` file and injects a quality challenge before the write lands.

`<target-repo>/.claude/hooks/expert-gate.sh`:

```bash
#!/usr/bin/env bash
# Quality gate for expert file writes. Challenges the agent to justify every addition
# against the signal bar before the write lands on disk.
set -euo pipefail

PAYLOAD=$(cat)
TOOL=$(echo "$PAYLOAD" | jq -r '.tool_name // ""')
FILE=""

if [[ "$TOOL" == "Write" ]]; then
  FILE=$(echo "$PAYLOAD" | jq -r '.tool_input.file_path // ""')
elif [[ "$TOOL" == "Edit" || "$TOOL" == "MultiEdit" ]]; then
  FILE=$(echo "$PAYLOAD" | jq -r '.tool_input.file_path // ""')
fi

# Only intercept expert/*.md files
[[ "$FILE" == */expert/*.md ]] || exit 0

echo "## Expert File Quality Gate"
echo "You are about to write to an expert file. Before proceeding, challenge every insight you are adding:"
echo "- Can it be found with a single grep or one file read? → do NOT add it"
echo "- Does understanding it require tracing multiple files, flows, or semantic layers? → belongs here"
echo "- Is it stable and reusable across different future tasks? → task-specific details do not belong here"
echo "Revise or discard anything that does not pass all three checks before writing."
```

### Hook 3 — session-compact-expert.sh (PostCompact)

Writes a marker when compaction occurs so the retrospective hooks know context has been lost and
stand down for the rest of the session.

`<target-repo>/.claude/hooks/session-compact-expert.sh`:

```bash
#!/usr/bin/env bash
# Records that context compaction has occurred this session.
# Retro hooks check for this marker and skip if found.
set -euo pipefail

PAYLOAD=$(cat)
SESSION_ID=$(echo "$PAYLOAD" | jq -r '.session_id // "unknown"')
touch "/tmp/expert-compacted-${SESSION_ID}"
```

### Hook 4 — session-monitor-expert.sh (PostToolUse)

Signals an expert retrospective when the context window reaches ~65% used — inside the healthy
zone, before context rot. Fires once per session; skips if compaction has already occurred.

`<target-repo>/.claude/hooks/session-monitor-expert.sh`:

```bash
#!/usr/bin/env bash
# Signals an expert retrospective when context window reaches ~60-65%.
# Fires once per session via a session-scoped state file.
# Skips if compaction has already occurred (context is no longer trustworthy).
set -euo pipefail

PAYLOAD=$(cat)
SESSION_ID=$(echo "$PAYLOAD" | jq -r '.session_id // "unknown"')
REMAINING_PCT=$(echo "$PAYLOAD" | jq -r '.context_window.remaining_pct // 100')

# Compaction occurred — context is lossy, skip retro
[[ -f "/tmp/expert-compacted-${SESSION_ID}" ]] && exit 0

STATE_FILE="/tmp/expert-signal-${SESSION_ID}"

[[ -f "$STATE_FILE" ]] && exit 0
# remaining_pct counts down from 100; trigger when 35% or less remains (= 65%+ used)
(( $(echo "$REMAINING_PCT > 35" | bc -l) )) && exit 0

touch "$STATE_FILE"
USED_PCT=$(echo "100 - $REMAINING_PCT" | bc)
echo "## Expert Retrospective Signal"
echo "Context is ~${USED_PCT}% full. Do a partial expert retrospective now while context is still crisp:"
echo "Ask what you have discovered that a future agent shouldn't have to rediscover, and update expert/index.md."
```

### Hook 5 — session-stop-expert.sh (PostToolUse: Bash)

Fires a retrospective after every git commit — whether made by Claude or by the user directly. The
git `post-commit` hook (Step 3b) writes a marker; this script reads and consumes it. Skips if the
65% retro already fired or if compaction has occurred.

`<target-repo>/.claude/hooks/session-stop-expert.sh`:

```bash
#!/usr/bin/env bash
# Triggers an expert retrospective after every git commit (Claude's or the user's).
# The marker is written by .git/hooks/post-commit; this script just consumes it.
# Fires as a PostToolUse/Bash hook; exits silently when no commit has occurred.
# Skips if the 65% context hook already ran a retro, or if compaction has occurred.
set -euo pipefail

PAYLOAD=$(cat)
SESSION_ID=$(echo "$PAYLOAD" | jq -r '.session_id // "unknown"')

REPO_ROOT="$(git -C "$(dirname "$0")" rev-parse --show-toplevel 2>/dev/null || echo "")"
REPO_HASH=$(echo "$REPO_ROOT" | md5 | cut -c1-8)
MARKER="/tmp/claude-expert-retrospective-${REPO_HASH}"

[[ -f "$MARKER" ]] || exit 0
rm -f "$MARKER"

# Compaction occurred — context is lossy, skip retro
[[ -f "/tmp/expert-compacted-${SESSION_ID}" ]] && exit 0

# 65% context hook already ran a retro this session — skip to avoid duplication
[[ -f "/tmp/expert-signal-${SESSION_ID}" ]] && exit 0

echo "## Expert Retrospective"
echo "A commit was just made. Ask: did I discover anything in this task that a future agent shouldn't have to rediscover?"
echo "If yes, update the relevant file in expert/ now. Only stable, reusable insights — not task history or one-off details."
```

Make all five scripts executable:

```
chmod +x <target-repo>/.claude/hooks/session-start-expert.sh
chmod +x <target-repo>/.claude/hooks/expert-gate.sh
chmod +x <target-repo>/.claude/hooks/session-compact-expert.sh
chmod +x <target-repo>/.claude/hooks/session-monitor-expert.sh
chmod +x <target-repo>/.claude/hooks/session-stop-expert.sh
```

### Step 3b — git post-commit hook

Create `<target-repo>/.git/hooks/post-commit` to write the marker file on every commit, including
commits the user makes outside Claude:

```bash
#!/usr/bin/env bash
# Signals Claude Code to run an expert retrospective after any commit.
REPO_HASH=$(git rev-parse --show-toplevel | md5 | cut -c1-8)
touch "/tmp/claude-expert-retrospective-${REPO_HASH}"
```

Make it executable:

```
chmod +x <target-repo>/.git/hooks/post-commit
```

Note: `.git/hooks/` is not tracked by git, so this file is local to the current clone. Mention this
to the user if relevant. If `.git/hooks/post-commit` already exists, append the marker line to it
rather than overwriting.

### Step 3c — settings.json

Add or update `<target-repo>/.claude/settings.json` to include:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/session-start-expert.sh" }]
      },
      {
        "matcher": "resume",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/session-start-expert.sh" }]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/expert-gate.sh" }]
      }
    ],
    "PostCompact": [
      {
        "matcher": ".*",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/session-compact-expert.sh" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": ".*",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/session-monitor-expert.sh" }]
      },
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/session-stop-expert.sh" }]
      }
    ]
  }
}
```

If `settings.json` already exists, **merge** these event keys into its existing `hooks` object — do
not overwrite other keys or other hook entries. If the file does not exist, create it with the full
structure above.

All hooks fire from inside the target repo regardless of where the agent is launched from. No
`CLAUDE.md` changes are needed.

---

## Step 4 — Remove obsolete soft directives (upgrade path)

If the repo already had the domain-expert system installed via soft directives in `CLAUDE.md` or
`AGENTS.md`, those instructions are now redundant — the hooks enforce the same behaviour
structurally. Remove them to avoid duplication.

### What to look for

Search `CLAUDE.md`, `AGENTS.md` (and any other agent instruction files) for lines that instruct
agents to read `expert/index.md` or to update `expert/` at end-of-task. Common forms:

- `**Before starting any task: read \`expert/index.md\`.**`
- `Mandatory read before starting any task: \`expert/index.md\``
- Any sentence containing both `expert/index.md` and "read" or "mandatory"

### What NOT to remove

- Substantive domain rules in `AGENTS.md` — only the "read expert/index.md" / "update expert/ at
  end of task" soft directives.
- Expert-file maintenance rules inside `expert/index.md` itself — those are part of the system's own
  documentation, not a soft directive.
- If a line mixes the expert directive with other content, remove only the expert-related part; keep
  the rest. If a section becomes empty after removal, remove the heading too.

After editing, re-read the affected files to confirm no directive remnants remain and no surrounding
content was accidentally damaged.

---

## Step 5 — Verify completion

Before declaring installation complete, confirm **all** of the following:

- [ ] `<target-repo>/expert/index.md` exists and contains the template content.
- [ ] `<target-repo>/.claude/hooks/session-start-expert.sh` exists and is executable.
- [ ] `<target-repo>/.claude/hooks/expert-gate.sh` exists and is executable.
- [ ] `<target-repo>/.claude/hooks/session-compact-expert.sh` exists and is executable.
- [ ] `<target-repo>/.claude/hooks/session-monitor-expert.sh` exists and is executable.
- [ ] `<target-repo>/.claude/hooks/session-stop-expert.sh` exists and is executable.
- [ ] `<target-repo>/.claude/settings.json` contains a `SessionStart` hook for both `startup` and `resume`.
- [ ] `<target-repo>/.claude/settings.json` contains a `PreToolUse` hook wired to `expert-gate.sh` for `Write|Edit|MultiEdit`.
- [ ] `<target-repo>/.claude/settings.json` contains a `PostCompact` hook wired to `session-compact-expert.sh`.
- [ ] `<target-repo>/.claude/settings.json` contains a `PostToolUse` hook wired to `session-monitor-expert.sh`.
- [ ] `<target-repo>/.claude/settings.json` contains a `PostToolUse/Bash` hook wired to `session-stop-expert.sh`.
- [ ] `<target-repo>/.git/hooks/post-commit` exists, is executable, and writes the marker file.
- [ ] (upgrade path) `CLAUDE.md` / `AGENTS.md` contain no remaining soft directives.

If any item is missing, fix it before finishing. Do not report success until the checklist is fully
satisfied. Then tell the user to restart their Claude Code session (or run `/hooks`) so the newly
wired hooks are loaded.

---

## Template: expert/index.md

```markdown
# Domain Experts

Purpose: save tokens for future agents by documenting domain knowledge that is important,
non-obvious, and expensive to reconstruct from code alone.

Every agent that reads or benefits from this folder must keep it up to date:
- remove or rewrite obsolete facts
- add new useful domain facts when they are stable and non-obvious
- keep entries concise and high-signal

Boundaries:
- include only important facts that usually require tracing multiple files, flows, or semantic layers
- do not restate code that is obvious from one file read or one grep
- do not turn this folder into general documentation, task history, or changelog notes

When to read this index:
- always — read expert/index.md at the start of every task, including operational, tooling, and
  non-codebase tasks; there are no exceptions for the index itself
- treat the index as a fixed, small context fee; do not skip it by reasoning that the task "isn't
  really about the code"

When not to read individual domain files (the index is still always read):
- narrow local edits whose ownership is already obvious from nearby code
- tasks where no domain file topic clearly overlaps with the work
- keep context as small as possible — read at most one domain file per task

Each expert file (expert/<domain>.md) must contain only:
- high-level architecture of the domain
- the files that actually own the domain, and what each file contributes
- non-obvious, high-value deductions that usually require tracing multiple files, flows, or semantic layers

Do not include:
- direct restatements of code that are obvious from a quick read
- broad project documentation or architecture overviews
- task history, bug diaries, or changelog notes — unless they reveal a stable domain invariant

Retrospective (mandatory at end of every non-trivial task):
Ask: "What did I need to explore in this session that a future agent shouldn't have to?"
If the answer is something important AND non-obvious that cannot be inferred from a single grep
or standard framework conventions, update the relevant expert file (or create a new one).
Rules:
- only add insights useful to future agents working on the same domain but a different task
- do not add insights that are too narrow or task-specific
- remove or rewrite obsolete facts
- keep all entries concise and high-signal

Index maintenance:
- add a link whenever a new domain file is created: `- [Domain Name](./filename.md)`
- remove the link when a domain file is deleted

## Entries
(none yet — add domain files as you discover non-obvious knowledge)
```
