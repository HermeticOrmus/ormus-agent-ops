---
name: agent-ops
description: Operational actions on tasks in an autonomous coding agent — stop, requeue, dead-letter, retry-subtasks, bump-priority. The mechanic's toolbox for running agents.
---

# /agent-ops — Operational Actions

> The verbs you need when something needs to be done to a task right now.

## Prerequisites

| Var | Purpose |
|---|---|
| `AGENT_API_URL` | Base URL of the agent's HTTP API |
| `AGENT_API_TOKEN` | Bearer token |

## Subcommands

### `/agent-ops stop <task-id>`

Stop a currently-running task. The task should transition to `stopped` (or whatever your agent's stopped state is named).

```bash
curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/stop"
```

When to use:
- Task is in a runaway loop
- Task is doing the wrong thing (operator changed mind)
- Task is consuming excessive cost / time

### `/agent-ops requeue <task-id>`

Move a `failed` or `stopped` task back to `queued` so a worker picks it up again.

```bash
curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/requeue"
```

When to use:
- Failure was infrastructure (`/agent-triage` classified as `infra`)
- Task was stopped to prioritize something else, now ready to resume
- Operator fixed a credential / env issue and wants to retry

### `/agent-ops dead-letter <task-id> [reason]`

Move a task to dead-letter status. It won't be retried automatically. Optional reason logged for audit.

```bash
curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reason": "<reason>"}' \
  "$AGENT_API_URL/api/tasks/<task-id>/dead-letter"
```

When to use:
- Task is wontfix (scope changed, no longer needed)
- Task has hit max retries with no progress
- Task is fundamentally broken in a way operator action can't fix

### `/agent-ops retry-subtasks <task-id>`

If your agent decomposes tasks into subtasks, retry only the failed subtasks of a parent (don't restart the whole tree).

```bash
curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/retry-subtasks"
```

When to use:
- A 5-subtask plan failed at subtask 3; subtasks 1-2 succeeded
- Don't want to redo the work that already passed
- The retry endpoint should re-examine pending/failed subtasks only

### `/agent-ops priority <task-id> <priority>`

Bump or lower a task's queue priority. Useful when the user/operator decides this task should jump the line.

```bash
curl -s -X PATCH -H "Authorization: Bearer $AGENT_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"priority": <priority>}' \
  "$AGENT_API_URL/api/tasks/<task-id>"
```

Priority semantics depend on your agent. Common conventions:
- Lower number = higher priority (1 = highest, 10 = lowest), OR
- Higher number = higher priority

Document which your agent uses in your env / runbook.

### `/agent-ops cancel <task-id> [reason]`

Like dead-letter but for `queued` (not yet running) tasks. Removes from queue without ever executing.

```bash
curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
  -d '{"reason": "<reason>"}' \
  "$AGENT_API_URL/api/tasks/<task-id>/cancel"
```

## Bulk variant

Each subcommand accepts multiple task IDs:

```
/agent-ops requeue task-1 task-2 task-3
/agent-ops dead-letter task-4 task-5 "scope changed"
```

Iterate, but **always confirm count + IDs before executing bulk operations**. A bulk dead-letter on the wrong list deletes work.

## Output

After each action, confirm with:

```
ACTION    <verb>
TASK      <task-id>
NEW STATE <state>
RESULT    ✓ ok / ✗ <error>
```

For bulk:

```
BULK ACTION  <verb> on <N> tasks
SUCCESS      <N>
FAILED       <N>
  • <task-id> — <error>
```

## Notes

- **Never run a bulk action without listing the IDs first.** Confirm the user wants exactly those IDs to receive the action.
- For irreversible actions (`dead-letter`, `cancel` with state cleanup), require explicit user confirmation even for single tasks.
- If your agent's API doesn't expose all these endpoints, omit the subcommands you don't need. The skill is a menu, not a contract.
- After running operational actions, suggest `/agent-status` to verify the new state.
