---
name: agent-evolve
description: Full-cycle improvement loop for an autonomous coding agent. Verify operational health, analyze pipeline state, research cutting-edge agent patterns, propose and apply improvements. The self-upgrading loop.
---

# /agent-evolve — Agent Self-Improvement Loop

> The agent that runs the agents needs a meta-loop: am I getting better, or am I just running?

## What it does

Periodically — weekly, monthly, after a notable failure pattern — run this loop to identify and apply improvements to the agent's prompts, configuration, and operational guardrails.

Six phases:

1. **Health verify** — is the agent actually working? `/agent-status` first.
2. **Pipeline analysis** — what does failure data over the last window reveal?
3. **Pattern research** — what's the current state of the art for autonomous coding agents?
4. **Proposal** — what specific changes (to prompts, config, guardrails) would move the metrics?
5. **Apply** — make the changes (with operator approval).
6. **Measure** — track whether the changes moved the metrics.

## Prerequisites

| Var | Purpose |
|---|---|
| `AGENT_API_URL` | Base URL of the agent's HTTP API |
| `AGENT_API_TOKEN` | Bearer token |
| `AGENT_REPO_PATH` | Local path to the agent's source code (so changes can be applied via git) |

## Phase 1: Health verify

Run `/agent-status` first. If the agent isn't healthy, **stop here**. Don't try to evolve a broken agent — fix it first.

## Phase 2: Pipeline analysis

Pull data for the analysis window (default: last 7 days):

```bash
# All tasks in window
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks?since=7d"
```

Compute:

| Metric | Why it matters |
|---|---|
| Fail rate | Trend up / down / flat |
| Time to first response per task | Latency trend |
| Time to completion per task | Throughput trend |
| Cost per task | Spend efficiency trend |
| Top 5 failure classes | Where to focus improvements |
| Review pass rate | Quality of generated code |
| Retry rate | Brittleness signal |
| Tasks that hit max-iterations | Prompt-engineering signal |

For each metric, compare to the previous window. Flag anything that regressed.

## Phase 3: Pattern research

Research recent advances in:

- Prompt engineering for coding agents (system prompts, planning patterns, recovery patterns)
- Tool use patterns (which tools the agent uses, in what order)
- Review pipelines (single-pass vs multi-pass review, model choice)
- Failure recovery (retry strategies, prompt-rewriting on failure)
- Cost optimization (model routing, caching, context compaction)

Sources:
- arxiv.org for recent papers (search "agent" + "code generation" + recent year)
- Major lab blogs (Anthropic, OpenAI, DeepMind) for post-mortems and best practices
- Open-source agent frameworks for prompt patterns
- The agent's own dead-letter pile for what's actually breaking

Don't research blindly — research with a hypothesis. "Our retry rate is up 30% — what does the research say about retry strategies?" beats "what's new in agents."

## Phase 4: Proposal

For each proposed improvement, document:

```markdown
## Proposal: <name>

**Hypothesis**: <what metric will move and why>
**Current**: <metric value>
**Target**: <metric value>
**Change**:
  - File: <path>
  - Diff: <unified diff or precise description>
**Rollback**: <how to revert if metrics get worse>
**Confidence**: <high / medium / low>
**Time to validate**: <how long before we know if it worked>
```

A typical evolve cycle produces 1-3 proposals, not 10. Quality over quantity. Each proposal should target a specific metric with a specific change.

## Phase 5: Apply (with operator approval)

```bash
cd "$AGENT_REPO_PATH"
git checkout -b evolve-<YYYY-MM-DD>
# Apply changes (Edit tool, etc.)
git add -A
git commit -m "evolve: <summary of changes>"

# Either push for review, or merge directly to main if you're the only operator
git push origin evolve-<YYYY-MM-DD>
# OR
git checkout main && git merge evolve-<YYYY-MM-DD> && git push
```

If your agent reloads from a deployment script, run that:

```bash
# Example: rsync + restart pattern
./scripts/deploy.sh
```

## Phase 6: Measure

Set a follow-up date (a week / a month from the cycle).

```
EVOLVE CYCLE  <ID>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Date:           <today>
Window:         <7d / 30d>
Health:         <green / yellow>

KEY METRICS (current vs previous window)
  fail_rate:        <Y%> (was <X%>)
  retry_rate:       <Y%> (was <X%>)
  time_to_complete: <Ymin> (was <Xmin>)
  cost_per_task:    $<Y>  (was $<X>)
  review_pass:      <Y%> (was <X%>)

CHANGES APPLIED  (<N>)
  • <change 1> — targeting <metric> — confidence <H/M/L>
  • <change 2> — targeting <metric> — confidence <H/M/L>

NEXT REVIEW    <date>
  Re-run /agent-evolve and compare metrics. If regressed, rollback the change(s).
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Anti-patterns to avoid

- **Don't apply 10 changes at once.** You won't know which one helped or hurt.
- **Don't evolve without metrics.** "Feels better" isn't a measurement.
- **Don't research without a hypothesis.** Open-ended research produces noise.
- **Don't skip Phase 1.** A broken agent looks the same as a failing one in the metrics — fix the break first.
- **Don't over-evolve.** Two cycles per month is plenty. Daily evolution is just thrashing.

## Cadence

- After a notable incident (e.g., 3+ tasks failed the same way) — within a week.
- Routine cycle: monthly.
- Keep a log of every cycle (cycle ID, changes applied, metrics before/after) so you can see the trend over time.

## Notes

- This skill is the meta-loop. It assumes the agent itself is built and operational. Use `/agent-status`, `/agent-triage`, `/agent-clean`, `/agent-inspect`, `/agent-ops`, `/agent-sweep-orphans` for the day-to-day operations; this skill is for the strategic improvement layer.
- The output of /agent-evolve is a small number of focused, measurable proposals. If your cycle produces zero proposals, that's a valid outcome — it means the agent is currently stable and there's no high-confidence improvement to make. Don't force changes.
