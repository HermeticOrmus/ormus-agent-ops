---
name: agent-inspect
description: Deep dive into a single task in an autonomous coding agent — details, log, events, dependencies, review verdict. Use when a task's behavior is mysterious or `/agent-triage` couldn't classify it.
---

# /agent-inspect — Deep Task Inspection

> When a task is weird, this is where you go.

## Prerequisites

| Var | Purpose |
|---|---|
| `AGENT_API_URL` | Base URL of the agent's HTTP API |
| `AGENT_API_TOKEN` | Bearer token |

Endpoints used (adapt to your agent's API):

- `GET /api/tasks/:id` — task metadata
- `GET /api/tasks/:id/log` — execution log
- `GET /api/tasks/:id/events` — event stream
- `GET /api/tasks/:id/review` — review verdict (if your agent runs review)
- `GET /api/tasks/:id/deps` — task dependencies / dependents

## Usage

```
/agent-inspect <task-id>
```

## Workflow

### 1. Pull metadata

```bash
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>" | jq
```

Capture: status, prompt, model, project, created_at, updated_at, attempt count, parent task (if subtask), priority, cost.

### 2. Pull events timeline

```bash
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/events" | jq
```

Build a timeline. Look for:
- State transitions (queued → running → review → done/failed)
- Retry attempts
- Worker reassignments
- Review pass/fail rounds
- External events (was the agent restarted? did the worker die?)

### 3. Pull log

```bash
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/log"
```

Read the **last 100 lines** for failures. For successful but mysterious tasks, scan for unusual patterns: long pauses, retries, tool errors, low-confidence verdicts.

### 4. Pull review verdict (if applicable)

If your agent runs review on output:

```bash
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/review"
```

Capture: verdict (PASS/WARN/FAIL/NEEDS_REVIEW), reviewer model, specific issues raised.

### 5. Pull dependency graph

```bash
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/deps"
```

Identify:
- Parent task (if this is a subtask)
- Sibling subtasks (other subtasks of the same parent)
- Dependent tasks (what's waiting on this one)

## Output format

```
TASK INSPECTION — <task-id>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

METADATA
  status:     <state>
  project:    <project>
  model:      <model>
  attempt:    <N> of <max>
  created:    <relative time>
  updated:    <relative time>
  duration:   <total / current>
  cost:       $<amount>
  parent:     <parent-task-id or "—">

PROMPT
  <first 200 chars of prompt; truncate if longer>

TIMELINE
  <ISO time>  queued
  <ISO time>  running (worker <N>)
  <ISO time>  review-requested
  <ISO time>  review-failed (<reason>)
  <ISO time>  retried
  <ISO time>  failed

REVIEW VERDICT
  verdict:    <PASS/WARN/FAIL/NEEDS_REVIEW>
  reviewer:   <model>
  issues:
    • <issue 1>
    • <issue 2>

DEPENDENCIES
  parent:     <parent or "—">
  siblings:   <count>  (passed: <N>, failed: <N>, pending: <N>)
  dependents: <count>  (waiting on this task)

LOG TAIL (last 30 lines)
  [paste last 30 lines, with line numbers]

DIAGNOSIS
  [1-3 sentences synthesizing what likely happened, based on timeline + log + verdict]

RECOMMENDED ACTION
  [specific next step: requeue / dead-letter / amend prompt / fix env / inspect parent]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Diagnostic patterns

| Symptom | Likely cause | Action |
|---|---|---|
| Task stuck in `running` for > expected time | Worker hung or zombie | Stop + requeue |
| Multiple `retried` events but same failure | Same root cause not fixed | Read log carefully, address cause |
| Review keeps failing with same complaint | Prompt under-specified | Amend prompt, requeue |
| Subtask succeeds but parent fails | Aggregation bug or last subtask broken | Inspect last subtask |
| All siblings passed, this one failed | Task-specific issue, not systemic | Look at this task's prompt vs siblings |

## Notes

- **Don't truncate the log** until you've read it. A 500-line log with the answer in line 247 means truncation lies.
- For very long logs (>2000 lines), summarize structurally (count of each event type) and quote only the last 30 lines verbatim.
- If diagnosis is uncertain, say so. Don't fabricate a cause.
