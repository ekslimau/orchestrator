---
name: Orchestrate Delegation
description: Split work into tasks and delegate to planner, coder, reviewer, or explorer. Use when leading a team of OpenCode subagents, choosing which worker to launch, writing a self-contained subagent task, running workers in parallel, or continuing an existing subagent session with a sessionID.
---

# Orchestrate Delegation

Playbook for a lead agent that coordinates subagents. The lead plans, delegates, reads results, and reports. It never edits files, never runs shell commands, and never calls itself.

## Core rules

1. **Decompose first.** Turn the request into concrete tasks with a clear done condition before launching anyone.
2. **One task, one worker, one self-contained prompt.** A child session starts with fresh context and cannot see the parent conversation. Include every file path, goal, constraint, and output format it needs.
3. **Integrate results.** Read each result and fold it into one final answer. Do not just forward raw output.
4. **Stay in your lane.** The lead coordinates. Editing and shell work belongs to the coder.

## Choosing a worker

| Need | Worker |
|---|---|
| Design, decide, sequence steps | `planner` |
| Read/write/fix code, run commands | `coder` |
| Verify correctness, security, tests | `reviewer` |
| Find files, symbols, patterns | `explorer` |

Read-only work (planning, review, search) can run in parallel. Never run two workers that write the same files at once.

## Subagent tool arguments

```
agent        (required)  planner | coder | reviewer | explorer
description  (required)  short UI label
prompt       (required)  full, self-contained task
model        (optional)  override the model
sessionID    (optional)  continue an existing subagent session
background   (optional)  run async and return immediately
```

### Start new work

Omit `sessionID`. This spawns a fresh child session and the result carries a new `sessionID`.

```
agent: explorer
description: Find config files
prompt: Search the repository at <path> for every file named *.config.*. Report each match with its absolute path. If none exist, say so explicitly.
```

### Continue an existing session

Pass the **exact** `sessionID` returned by a previous subagent call. Use this when the follow-up needs the child's prior context.

```
agent: explorer
description: Find test files
prompt: Using the same repository, now list every *.test.* file.
sessionID: ses_ec543f6f8ffegxASYWJcI7Rv24
```

### The one rule that bites

**Never invent a sessionID.** Placeholders such as `ses_invalid`, `ses_invalid2`, or a made-up string are rejected and the whole call fails:

```
Subagent session not found: ses_invalid
```

If you do not hold a real sessionID from an earlier result, start a fresh call without one.

## Writing a good task prompt

A child has no memory of the parent thread. A complete prompt answers:

- **Where** — absolute paths and the working directory.
- **What** — the exact goal and its done condition.
- **Constraints** — files that are read-only, sizes or line limits, style to follow.
- **Output** — the format you want back (a list of paths, a diff summary, findings by severity).

Weak: `Look at the auth code.`
Strong: `Read src/auth/session.ts and list every place the token expiry is compared. Report each with line number and the exact comparison. Do not edit anything.`

## Parallel delegation

Launch independent, non-overlapping tasks together. The five parallel explores in the reference run completed in about 35 ms each because they shared no files.

Keep sequential what depends on a prior result — for example, "continue the explorer session that found the modules" needs the first call's sessionID.

## Reporting

End with the integrated answer, not a transcript:

- The result the user asked for.
- Any blocker, with what is needed to unblock it.
- A one-line note on what was skipped and the risk.
