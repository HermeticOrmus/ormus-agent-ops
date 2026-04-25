---
name: agent-triage
description: Classify failed tasks from an autonomous coding agent, recommend disposition (retry / dead-letter / fix-and-retry / wontfix), and offer bulk action. Replaces reading hundreds of failure logs by hand.
---

# /agent-triage — Smart Failure Triage

> Failed tasks fall into a small number of buckets. Classify, then act.

## Prerequisites

| Var | Purpose |
|---|---|
| `AGENT_API_URL` | Base URL of the agent's HTTP API |
| `AGENT_API_TOKEN` | Bearer token |

Endpoints used:
- `GET /api/tasks?status=failed&since=<window>` — list failed tasks
- `GET /api/tasks/:id/log` — full log for one task
- `POST /api/tasks/:id/requeue` — requeue
- `POST /api/tasks/:id/dead-letter` — dead-letter

## Workflow

### Phase 1: Pull failures

```bash
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks?status=failed&since=24h"
```

### Phase 2: Classify each failure

For each failed task, fetch its log/last error and assign one classification:

| Class | Pattern | Recommended action |
|---|---|---|
| `infra` | API rate limit, credit exhausted, network timeout, 5xx from upstream | **requeue** — not the task's fault |
| `auth` | 401/403 from upstream, expired token, missing credential | **fix-and-retry** — operator must rotate or refresh |
| `code` | Compile error, test failure, lint failure that the agent could fix | **requeue** with prompt amendment, or **dead-letter** if 3+ retries already |
| `prompt` | Agent looped, hit max iterations without converging, hallucinated nonexistent file | **fix-and-retry** — operator amends the prompt |
| `env` | Missing dep, wrong Node version, path doesn't exist on worker | **fix-and-retry** — operator fixes the env |
| `wontfix` | Task is no longer needed, superseded by another, or scope was wrong | **dead-letter** |
| `unknown` | Doesn't match any pattern above | **inspect** — drill in with `/agent-inspect <task-id>` |

### Phase 3: Present grouped summary

```
TRIAGE — <N failures> in last 24h
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

INFRA           <count>  → bulk requeue
  • <task-id> <slug>  — <one-line cause>
  • <task-id> <slug>  — <one-line cause>

AUTH            <count>  → fix credentials, then requeue
  • <task-id> <slug>  — <missing/expired creds>

CODE            <count>  → requeue with amendment OR dead-letter
  • <task-id> <slug>  — <error type>  — <attempt N/3>

PROMPT          <count>  → operator amends prompt
  • <task-id> <slug>  — <pattern: looped / hallucinated / max-iter>

ENV             <count>  → fix worker env, then requeue
  • <task-id> <slug>  — <missing dep / wrong version>

WONTFIX         <count>  → dead-letter
  • <task-id> <slug>  — <reason>

UNKNOWN         <count>  → /agent-inspect each
  • <task-id> <slug>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PROPOSED ACTIONS
  Bulk requeue:    <N> tasks (infra)
  Bulk dead-letter: <N> tasks (wontfix)
  Operator action:  <N> tasks (auth/prompt/env)
  Inspect deeper:   <N> tasks (unknown)
```

### Phase 4: Bulk action

After presenting, offer bulk operations. **Do NOT execute without explicit user approval.**

```bash
# Bulk requeue infra failures (after user confirms)
for tid in $INFRA_TASK_IDS; do
  curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
    "$AGENT_API_URL/api/tasks/$tid/requeue"
done
```

## Heuristics for log → classification

Common patterns to grep in logs:

| Pattern | Class |
|---|---|
| `rate.limit\|429\|credit.balance.exhausted\|insufficient.credits` | infra |
| `401\|403\|invalid.api.key\|expired.token\|unauthor` | auth |
| `Module not found\|Cannot find package\|node_modules` | env |
| `iteration.limit\|max.iterations\|loop.detected\|too many turns` | prompt |
| `ENOENT\|file.not.found\|path.does.not.exist` | env (probably) |
| `tsc.*error\|test.*failed\|lint` | code |
| `superseded\|cancelled.by.user\|scope.changed` | wontfix |

When in doubt, classify as `unknown` and recommend `/agent-inspect`.

## Notes

- Default window is 24h. For longer windows (`/agent-triage 7d`), sweep is heavier — paginate accordingly.
- Don't auto-classify based on task slug or project name — read the actual error.
- A single task may match multiple patterns. Pick the most specific (auth beats code, env beats prompt).
- **Never bulk-act without showing the user the count + class breakdown first.** A misclassification that bulk-requeues 50 prompt failures will create 50 more failures.
