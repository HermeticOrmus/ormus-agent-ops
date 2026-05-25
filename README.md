<p align="center">
  <img src="https://ormus.solutions/mascot/chain_braces_to_swan.gif" alt="ormus-agent-ops" width="128" style="image-rendering: pixelated;" />
</p>

<h1 align="center">ormus-agent-ops</h1>

<p align="center">
  <em>Operator's playbook for autonomous coding agents. 7 Claude Code skills: status, triage, ops, inspect, sweep, clean, evolve.</em>
</p>

<p align="center">
  <a href="https://github.com/HermeticOrmus/ormus-agent-ops/stargazers"><img src="https://img.shields.io/github/stars/HermeticOrmus/ormus-agent-ops?style=flat-square&color=aa8142" alt="Stars" /></a>
  <a href="https://github.com/HermeticOrmus/ormus-agent-ops/blob/main/LICENSE"><img src="https://img.shields.io/github/license/HermeticOrmus/ormus-agent-ops?style=flat-square&color=aa8142" alt="License" /></a>
  <a href="https://github.com/HermeticOrmus/ormus-agent-ops/commits"><img src="https://img.shields.io/github/last-commit/HermeticOrmus/ormus-agent-ops?style=flat-square&color=aa8142" alt="Last Commit" /></a>
  <img src="https://img.shields.io/badge/Claude_Code-aa8142?style=flat-square&logo=anthropic&logoColor=white" alt="Claude Code" />
</p>

---

> **Operator's playbook for running autonomous coding agents. 7 Claude Code skills bundled as a marketplace plugin: **status**, **triage**, **ops**, **inspect**, **sweep-orphans**, **clean**, **evolve**.**

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

---

## Part of the Libre Open-Source Stack for Claude Code

This repository is part of a growing family of open-source toolkits for Claude Code.

### Libre suite — comprehensive plugin bundles

- [LibreUIUX-Claude-Code](https://github.com/HermeticOrmus/LibreUIUX-Claude-Code) — UI/UX development (152 agents, 70 plugins, 76 commands, 74 skills)
- [LibreArch-Claude-Code](https://github.com/HermeticOrmus/LibreArch-Claude-Code) — Software architecture and system design
- [LibreCopy-Claude-Code](https://github.com/HermeticOrmus/LibreCopy-Claude-Code) — Technical writing and documentation engineering
- [LibreDevOps-Claude-Code](https://github.com/HermeticOrmus/LibreDevOps-Claude-Code) — DevOps engineering and infrastructure automation
- [LibreEmbed-Claude-Code](https://github.com/HermeticOrmus/LibreEmbed-Claude-Code) — Embedded systems, firmware, and IoT development
- [LibreFinTech-Claude-Code](https://github.com/HermeticOrmus/LibreFinTech-Claude-Code) — Financial technology development
- [LibreGEO-Claude-Code](https://github.com/HermeticOrmus/LibreGEO-Claude-Code) — AI-search optimization (ChatGPT, Perplexity, Gemini, Google AI Overviews)
- [LibreGameDev-Claude-Code](https://github.com/HermeticOrmus/LibreGameDev-Claude-Code) — Game development across Godot, Unity, Unreal
- [LibreMLOps-Claude-Code](https://github.com/HermeticOrmus/LibreMLOps-Claude-Code) — ML engineering and AI operations
- [LibreMobileDev-Claude-Code](https://github.com/HermeticOrmus/LibreMobileDev-Claude-Code) — Mobile app development (Flutter, React Native, native iOS, native Android)
- [LibreSecOps-Claude-Code](https://github.com/HermeticOrmus/LibreSecOps-Claude-Code) — Security operations

### Skills mini-repos — single CLAUDE.md drop-ins

- [vibe-engineer-skills](https://github.com/HermeticOrmus/vibe-engineer-skills) — Direct AI codegen well (hypothesis → scope → validate → reject working-but-wrong)
- [markdown-discipline-skills](https://github.com/HermeticOrmus/markdown-discipline-skills) — Strip AI-slop from markdown (no em dashes, no marketing fluff)
- [shell-safety-skills](https://github.com/HermeticOrmus/shell-safety-skills) — `set -euo pipefail` discipline + 15 failure-mode examples
- [commit-standard-skills](https://github.com/HermeticOrmus/commit-standard-skills) — Ormus Commit Standard v1.0 + commit-msg hook + commitlint
- [unwoke-skills](https://github.com/HermeticOrmus/unwoke-skills) — Strip AI theater (ten sins to eliminate, symmetric engagement)
- [python-conventions-skills](https://github.com/HermeticOrmus/python-conventions-skills) — Modern Python 3.11+ (types, pathlib, async, ruff, mypy, uv)
- [typescript-conventions-skills](https://github.com/HermeticOrmus/typescript-conventions-skills) — TypeScript strict mode, discriminated unions, Result types
- [hermetic-laws-skills](https://github.com/HermeticOrmus/hermetic-laws-skills) — Seven Hermetic Principles applied to engineering
- [riper-workflow-skills](https://github.com/HermeticOrmus/riper-workflow-skills) — Research / Innovate / Plan / Execute / Review systematic dev
- [six-day-cycle-skills](https://github.com/HermeticOrmus/six-day-cycle-skills) — Sustainable shipping cadence with mandatory rest
- [token-optimization-skills](https://github.com/HermeticOrmus/token-optimization-skills) — Claude Code token + context optimization
- [osint-skills](https://github.com/HermeticOrmus/osint-skills) — OSINT research methodology (multi-wave investigative spiral)
- [calcinate-skills](https://github.com/HermeticOrmus/calcinate-skills) — Stage 1 of the Magnum Opus (burn project bloat)
- [claude-md-overhaul-skills](https://github.com/HermeticOrmus/claude-md-overhaul-skills) — Audit CLAUDE.md and MEMORY.md against caps
- [session-handoff-skills](https://github.com/HermeticOrmus/session-handoff-skills) — Session handoff + pickup discipline
- [naming-skills](https://github.com/HermeticOrmus/naming-skills) — Product naming methodology (mine the brand's vocabulary)
- [magnum-opus-skills](https://github.com/HermeticOrmus/magnum-opus-skills) — Seven-stage alchemy applied to project transformation

### Template source

- [andrej-karpathy-skills](https://github.com/HermeticOrmus/andrej-karpathy-skills) — the canonical single-file CLAUDE.md pattern (fork of jiayuan_jy's original)

Star the family, not just one — that's how the suite stays coherent.
