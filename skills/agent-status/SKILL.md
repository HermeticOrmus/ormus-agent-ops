---
name: agent-status
description: At-a-glance overview of an autonomous coding agent — health, running tasks, queue, fail rate. Use to answer "is the agent working and what's it doing?" in a single dashboard.
---

# /agent-status — Agent Status Dashboard

> One screen. Health, queue, throughput, recent failures. No spelunking.

## Prerequisites

Set these env vars (or pass via shell when invoking):

| Var | Purpose |
|---|---|
| `AGENT_API_URL` | Base URL of the agent's HTTP API (e.g. `https://agent.example.com` or `http://localhost:7777`) |
| `AGENT_API_TOKEN` | Bearer token for authenticated endpoints |

The skill expects a task-queue-shaped agent that exposes endpoints roughly matching:

- `GET /api/health` — health summary
- `GET /api/tasks?status=running` — running tasks
- `GET /api/tasks?status=queued` — queued tasks
- `GET /api/tasks?status=failed&since=24h` — recent failures

If your agent exposes different endpoints, adapt the curl invocations below.

## When invoked

Pull a single dashboard frame answering:

1. **Health** — service up? DB reachable? Workers responsive?
2. **Active work** — what's running right now (count + brief task summaries)
3. **Queue** — how many queued, how long is the oldest
4. **Throughput** — tasks completed in the last hour / 24h
5. **Failures** — fail rate over last 24h, top 3 most recent failures
6. **Cost** — if the agent tracks per-task spend, sum last 24h

## Workflow

```bash
# 1. Health
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" "$AGENT_API_URL/api/health"

# 2. Running tasks
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" "$AGENT_API_URL/api/tasks?status=running"

# 3. Queue depth + oldest
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" "$AGENT_API_URL/api/tasks?status=queued&order=created_at"

# 4. Recent failures
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" "$AGENT_API_URL/api/tasks?status=failed&since=24h"
```

## Output format

```
AGENT STATUS — <ISO timestamp>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

HEALTH    [✓ healthy | ⚠ degraded | ✗ down]
  service: <up/down>
  workers: <N/M alive>
  db: <reachable/error>

ACTIVE    <N running>
  • <task-id> <slug> · running <duration>
  • <task-id> <slug> · running <duration>
  • <task-id> <slug> · running <duration>

QUEUE     <N queued | oldest <duration> ago>

LAST 24H  <N completed | <N failed> | <X.X% fail rate>

FAILURES  (last 3)
  • <task-id> <slug> — <error class> — <duration> ago
  • <task-id> <slug> — <error class> — <duration> ago
  • <task-id> <slug> — <error class> — <duration> ago

COST      $<amount> spent / 24h (if tracked)

NEXT      Recommended action: <triage failures / clear queue / nothing>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## When to recommend follow-up

- **Fail rate >5% over last 24h** → suggest `/agent-triage`
- **Queue depth >20 with oldest >1h** → suggest investigation (worker stuck?)
- **Health degraded** → suggest checking logs / restart
- **Workers <expected count** → suggest checking process supervisor

## Notes

- The skill assumes a single agent. For multi-agent setups, set `AGENT_API_URL` per-invocation or run the skill once per agent.
- Don't paginate failures here — that's `/agent-triage`'s job. Show only the top 3.
- Keep the dashboard one screen. If anything overflows, summarize and suggest the deeper-dive skill.
