---
name: agent-clean
description: Purge tasks from an autonomous coding agent that failed due to infrastructure errors (rate limits, credit exhaustion, auth failures, network) — not code failures. Cleans the queue without losing legitimate failures that need triage.
---

# /agent-clean — Infrastructure-Failure Purge

> Don't conflate "the agent didn't try because we ran out of credits" with "the agent tried and the code is broken."

## The signal

Failed tasks in an autonomous agent fall into two completely different categories:

1. **Infra failures** — agent never got a chance: rate limit, credit exhausted, network timeout, auth expired
2. **Real failures** — agent tried, code is broken, prompt was wrong, env was missing

Bulk-requeuing all failures conflates these. Bulk-dead-lettering all failures throws away real work. `/agent-clean` separates them: it purges (or requeues) only the infra failures and leaves the real failures for `/agent-triage`.

## Prerequisites

| Var | Purpose |
|---|---|
| `AGENT_API_URL` | Base URL of the agent's HTTP API |
| `AGENT_API_TOKEN` | Bearer token |

## Usage

```
/agent-clean                    — sweep last 24h
/agent-clean 7d                 — sweep last 7 days
/agent-clean --requeue          — requeue infra failures instead of dead-lettering
/agent-clean --dry-run          — show what would be done, don't act
```

## Workflow

### Phase 1: Pull failures

```bash
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks?status=failed&since=24h"
```

### Phase 2: Match each failure against infra patterns

For each failed task, fetch a tail of its log and match against these patterns:

| Pattern | Class |
|---|---|
| `rate.?limit` / `429 Too Many Requests` | `rate-limit` |
| `credit.?balance.?exhausted` / `insufficient.?(credits\|balance)` / `quota.?exceeded` | `credit-exhausted` |
| `401 Unauthorized` / `invalid.?api.?key` / `expired.?token` | `auth-expired` |
| `ECONNRESET` / `ETIMEDOUT` / `getaddrinfo.?(ENOTFOUND\|EAI_AGAIN)` / `socket hang up` | `network` |
| `500 Internal Server Error` / `502 Bad Gateway` / `503 Service Unavailable` (from upstream APIs) | `upstream-5xx` |
| `worker.?(died\|killed\|crashed)` | `worker-crash` |

Anything that doesn't match — leave alone. That's a real failure that needs `/agent-triage`.

### Phase 3: Group + present

```
INFRA SWEEP — <window>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total failures:           <N>
Infra failures:           <N>  ←  this skill cleans these
Real failures:            <N>  ←  use /agent-triage

INFRA BREAKDOWN
  rate-limit:        <N>
  credit-exhausted:  <N>
  auth-expired:      <N>
  network:           <N>
  upstream-5xx:      <N>
  worker-crash:      <N>

PROPOSED ACTION
  default:           dead-letter all infra failures (with reason logged)
  --requeue mode:    requeue all infra failures

Real failures NOT touched:  <N>  (run /agent-triage to handle these)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Confirm? [y/N]
```

### Phase 4: Execute

After explicit user confirmation:

```bash
# Default: dead-letter (cleaner — these were never the agent's fault, no need to retry)
for tid in $INFRA_TASK_IDS; do
  curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"reason\": \"infra: $CLASS\"}" \
    "$AGENT_API_URL/api/tasks/$tid/dead-letter"
done
```

Or with `--requeue`:

```bash
for tid in $INFRA_TASK_IDS; do
  curl -s -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
    "$AGENT_API_URL/api/tasks/$tid/requeue"
done
```

## When to requeue vs dead-letter

| Scenario | Action |
|---|---|
| Rate limit hit, but you've upgraded the API tier | `--requeue` |
| Credits exhausted, you've topped up | `--requeue` |
| Auth expired, you've rotated tokens | `--requeue` |
| Generic network blip 3 days ago, work is no longer needed | dead-letter (default) |
| Worker crashed during a task that's no longer relevant | dead-letter |
| Upstream API was down briefly, work IS still needed | `--requeue` |

When in doubt: **dead-letter** is safer (reversible — you can re-create the task). `--requeue` only when you've fixed the root cause.

## Pairs with

- **`/agent-triage`** — handles the *real* failures (code, prompt, env). Run `/agent-clean` first to remove noise, then `/agent-triage` on what remains.
- **`/agent-status`** — verify the queue is clean afterward.

## Notes

- **Don't auto-requeue rate-limit failures while still rate-limited.** You'll create another wave of failures. Verify the underlying constraint is fixed first.
- **Log the reason on dead-letter.** A future audit looking at "why was this dead-lettered" needs to see "infra: credit-exhausted" not just "dead-lettered."
- This skill is intentionally scoped narrowly. Don't try to clean code/prompt/env failures here — that's `/agent-triage`'s job. Keep the scope clean.
