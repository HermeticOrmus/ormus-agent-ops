# ormus-agent-ops

> Operator's playbook for running autonomous coding agents. 7 Claude Code skills bundled as a marketplace plugin: **status**, **triage**, **ops**, **inspect**, **sweep-orphans**, **clean**, **evolve**.

If you run an autonomous coding agent — one that takes tasks from a queue, runs them in workers, commits to branches, and reports back — you need an operator's toolbox. This is mine, generalized.

## Why

Most write-ups about autonomous agents focus on *building* the agent. Almost nothing talks about *running* one. After enough hours operating one in production, the same patterns emerged over and over:

- **Status** — what's running, what's queued, what's failing? One screen, no spelunking.
- **Triage** — failed tasks fall into a small number of buckets (infra / auth / code / prompt / env / wontfix). Classify, then act.
- **Ops** — verbs you need on a task: stop, requeue, dead-letter, retry-subtasks, bump-priority.
- **Inspect** — when a task is weird, deep-dive its events, log, review verdict, dependency graph.
- **Sweep orphans** — tasks marked "done" with branches that never merged. Bulk triage with disposition options.
- **Clean** — purge tasks failed by infrastructure errors (rate limit, credits, network) without losing real failures.
- **Evolve** — meta-loop: verify health, analyze metrics, research, propose, apply, measure.

That's the full operator playbook. These 7 skills cover the day-to-day plus the strategic improvement layer.

## Install

```bash
claude plugin marketplace add HermeticOrmus/ormus-agent-ops
```

Then enable the plugin from the marketplace UI. All 7 skills become available.

Or clone directly into your skills directory:

```bash
git clone https://github.com/HermeticOrmus/ormus-agent-ops
# Symlink each skill into ~/.claude/skills/
for s in agent-status agent-triage agent-ops agent-inspect agent-sweep-orphans agent-clean agent-evolve; do
  ln -sf "$(pwd)/ormus-agent-ops/skills/$s" ~/.claude/skills/$s
done
```

## Configuration

Set these env vars (or pass per-invocation):

| Var | Required | Purpose |
|---|---|---|
| `AGENT_API_URL` | yes | Base URL of your agent's HTTP API (e.g. `http://localhost:7777`) |
| `AGENT_API_TOKEN` | yes | Bearer token for authenticated endpoints |
| `AGENT_REPO_PATH` | for `/agent-evolve` | Local path to the agent's source code |
| `GH_TOKEN` | for `/agent-sweep-orphans` | If working with GitHub remotely |

## Expected agent API shape

The skills assume a task-queue-shaped agent that exposes endpoints roughly matching:

| Method | Path | Used by |
|---|---|---|
| `GET` | `/api/health` | status |
| `GET` | `/api/tasks?status=<state>&since=<window>&project=<name>` | status, triage, sweep, clean |
| `GET` | `/api/tasks/:id` | inspect |
| `GET` | `/api/tasks/:id/log` | triage, inspect, clean |
| `GET` | `/api/tasks/:id/events` | inspect |
| `GET` | `/api/tasks/:id/review` | inspect (if your agent runs review) |
| `GET` | `/api/tasks/:id/deps` | inspect |
| `POST` | `/api/tasks/:id/stop` | ops |
| `POST` | `/api/tasks/:id/requeue` | ops, triage, clean |
| `POST` | `/api/tasks/:id/dead-letter` | ops, triage, clean |
| `POST` | `/api/tasks/:id/cancel` | ops |
| `POST` | `/api/tasks/:id/retry-subtasks` | ops |
| `PATCH` | `/api/tasks/:id` | ops (priority changes) |

If your agent's API differs, the skills are **playbooks**, not implementations. Adapt the curl invocations in each `SKILL.md` to your endpoints. The workflows still apply.

## Skill index

| Skill | Use when |
|---|---|
| `/agent-status` | "Is the agent working? What's it doing right now?" |
| `/agent-triage` | "Several tasks failed in the last day — what kind of failures and what should I do?" |
| `/agent-ops` | "I need to stop / requeue / dead-letter / retry / reprioritize a specific task." |
| `/agent-inspect` | "This one task is acting weird and I need to understand why." |
| `/agent-sweep-orphans` | "We have a pile of done-but-never-merged feature branches — clean them up." |
| `/agent-clean` | "Yesterday's failures were mostly rate-limits / credit-exhaustion. Purge those without touching real failures." |
| `/agent-evolve` | "It's been a month. What can I improve about how this agent runs?" |

## Recommended cadence

- **`/agent-status`** — whenever you sit down to operate the agent. 30 seconds.
- **`/agent-triage`** — daily or every-other-day if there are recent failures. 5-15 minutes.
- **`/agent-clean`** — after any infra incident (rate limit, credit refill, auth rotation). 2 minutes.
- **`/agent-sweep-orphans`** — weekly per project. 15-30 minutes per project.
- **`/agent-evolve`** — monthly, or after a notable failure pattern emerges. 1-2 hours.

`/agent-ops` and `/agent-inspect` are reactive — invoke when needed.

## Pairs with

Other Ormus session-lifecycle skills:

- [ormus-handoff](https://github.com/HermeticOrmus/ormus-handoff) · [ormus-pickup](https://github.com/HermeticOrmus/ormus-pickup) · [ormus-absorb](https://github.com/HermeticOrmus/ormus-absorb) · [ormus-explore](https://github.com/HermeticOrmus/ormus-explore) · [ormus-vibe-proof](https://github.com/HermeticOrmus/ormus-vibe-proof) · [ormus-meta-prompting](https://github.com/HermeticOrmus/ormus-meta-prompting)

Self-hosted Ormus apps:

- [ormus-invoicer](https://github.com/HermeticOrmus/ormus-invoicer) · [ormus-polls](https://github.com/HermeticOrmus/ormus-polls) · [ormus-presentations](https://github.com/HermeticOrmus/ormus-presentations) · [ormus-analyst](https://github.com/HermeticOrmus/ormus-analyst)

## License

MIT. See [LICENSE](LICENSE).

## Origin

Distilled from real production operation of an autonomous coding agent over several months. Every skill maps to a recurring operational pattern that emerged from running the same actions hundreds of times. The classification taxonomy in `/agent-triage` and the orphan classes in `/agent-sweep-orphans` are empirical — those buckets covered every real-world case across many months of failures and orphans, no theoretical cases added.

The skills are deliberately **decoupled from any specific agent product**. They define the operator playbook in terms of a generic task-queue API. Use them with any agent — fork an existing autonomous coder, wire up the API endpoints described above, and these skills work.
