---
name: Subagent Troubleshooting
description: Diagnose failed subagent delegation in OpenCode, especially "Subagent session not found". Use when a delegation errors, when a subagent result is missing, when continuing a subagent session fails, or when the orchestrator refuses to start a worker.
---

# Subagent Troubleshooting

Fix delegation problems between a lead agent and its subagents. Work from the symptom to the cause.

## `Subagent session not found: <id>`

The call passed a `sessionID` that no session owns.

| Cause | Fix |
|---|---|
| Model invented an ID (`ses_invalid`, placeholder) | Omit `sessionID` to start fresh, or pass the exact ID from a previous result |
| ID copied wrong (truncated, extra characters) | Re-read the real `sessionID` from the earlier tool output |
| ID belongs to a different parent | A session can only continue a child of the current session; start fresh |
| Session was deleted or expired | Start fresh |

The reliable value is the `sessionID` inside the previous `subagent` result:

```
<subagent sessionID="ses_ec543f6f8ffegxASYWJcI7Rv24" state="completed">
```

### Why it keeps recurring in a session

A running session keeps the system prompt it started with. Editing `opencode.json` does **not** change a session already open. If the old prompt taught the model to guess IDs, that session keeps failing. **Start a new orchestrator session** to pick up the fixed prompt.

### Prevention

Put this in the orchestrator system prompt:

> The subagent tool arguments are agent, description, prompt, plus the optional model, sessionID, background. To start new work omit sessionID. To continue, pass the exact sessionID returned by a previous subagent call. Never invent or guess a sessionID.

## `Subagent denied: <agent>`

The parent's `subagent` permissions do not allow that agent. Add an allow rule in the parent's `permissions`:

```jsonc
{ "action": "subagent", "resource": "coder", "effect": "allow" }
```

Rules run in order and the last match wins. A broad `{"resource":"*","effect":"deny"}` must come **before** the specific allows.

## `Subagent depth limit reached`

A worker tried to launch its own subagent. Delegation is one-directional; each worker should carry `{"action":"subagent","resource":"*","effect":"deny"}`. If nesting is genuinely needed, raise `experimental.subagent_depth`.

## `Unknown agent: <id>`

The requested worker does not exist or is not visible. Check the spelling against the `agents` map and confirm the agent's `mode` is `subagent` or `all`.

## Worker runs but ignores its task

Give the child a self-contained prompt. It cannot see the parent conversation. Add absolute paths, the done condition, constraints, and the output format. See the **orchestrate-delegation** skill.

## Diagnostic checklist

1. Read the exact error string.
2. For a session error, compare the ID passed against real IDs from previous results.
3. For a permission error, read the parent agent's `permissions` array in order.
4. Confirm the worker exists and its `mode` allows subagent use.
5. If a running session misbehaves after a config fix, start a new session.

## Reference

- Config: `opencode.json`, `agents.<name>.permissions`, `experimental.subagent_depth`.
- Skill: **orchestrate-delegation** for how to delegate correctly.
