---
name: agent-sweep-orphans
description: Triage orphan tasks in an autonomous coding agent — tasks marked "done" with commits or branches that never merged to main. Classifies each, shows the diff, and offers bulk merge / dead-letter / requeue dispositions.
---

# /agent-sweep-orphans — Orphan Task Cleanup

> "Done" doesn't mean "shipped." This sweep finds the gap and closes it.

## The orphan problem

Autonomous agents that commit to feature branches (instead of pushing directly to `main`) accumulate orphans:

- Task ran, committed to `feature/<task-id>`, marked itself done
- The PR was never merged (review caught issues, scope changed, branch protection blocked it)
- Now there's a branch sitting in the repo with completed work that's NOT on `main`

After a few months: dozens of orphan branches, each with potentially valuable work. This skill triages them in bulk.

## Prerequisites

| Var | Purpose |
|---|---|
| `AGENT_API_URL` | Base URL of the agent's HTTP API |
| `AGENT_API_TOKEN` | Bearer token |

Plus local `git` access to the project repo, OR a GitHub token (`GH_TOKEN`) if working remotely.

## Workflow

### Phase 1: Identify orphans

For a single project:

```bash
# Pull all "done" tasks for this project
curl -s -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks?status=done&project=<project>"

# For each task, check if its branch is reachable from main
git fetch origin
for branch in $(git branch -r | grep "feature/<task-id>"); do
  if git merge-base --is-ancestor "$branch" origin/main; then
    echo "MERGED: $branch"
  else
    echo "ORPHAN: $branch"
  fi
done
```

### Phase 2: Classify each orphan

| Class | Test | Disposition |
|---|---|---|
| **mergeable** | `git merge-tree main feature/X` produces no conflicts AND tests pass | **merge** — fast-forward or squash-merge |
| **conflicting** | merge-tree produces conflicts | **resolve-and-merge** OR **requeue** to redo against current main |
| **superseded** | another task already implemented this functionality (search recent commits on main) | **dead-letter** — close the branch |
| **stale** | branch is >30 days old AND no longer relevant | **dead-letter** |
| **wrong-target** | branch was based on a different default branch (e.g. `develop` not `main`) | **rebase-and-merge** OR **requeue** |
| **partial** | branch has commits but task report shows incomplete work | **requeue** to finish the work |

### Phase 3: Present per-orphan triage

For each orphan, show:

```
ORPHAN  <task-id>  <slug>
─────────────────────────
branch:    feature/<task-id>
created:   <date>
class:     <mergeable / conflicting / superseded / stale / wrong-target / partial>
diff:      +<lines>/-<lines> across <N> files
files:     <top 5 changed files>
last commit: <hash> <date> <message>

DIFF SUMMARY
  <brief description of what changed>

PROPOSED ACTION  <merge / dead-letter / requeue / inspect>

[ apply | skip | inspect | requeue with notes ]
```

### Phase 4: Bulk disposition (after per-orphan review)

After the user has approved/declined each one, batch the actions:

```
BULK PLAN
  merge:        <N> branches  (will fast-forward if possible, else open PRs)
  dead-letter:  <N> tasks     (close branches, no merge)
  requeue:      <N> tasks     (redo work against current main)
  skip:         <N>            (leave for next sweep)

Confirm? [y/N]
```

Only execute after explicit user confirmation.

### Phase 5: Execute

For merges:

```bash
# Try fast-forward first; fall back to merge commit if non-FF
git checkout main
git pull origin main
git merge --ff-only feature/<task-id> || git merge feature/<task-id>
git push origin main
```

Or open PRs (if branch protection requires review):

```bash
gh pr create --base main --head feature/<task-id> --title "<task slug>" --body "..."
```

For dead-letter:

```bash
git push origin --delete feature/<task-id>
curl -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
  "$AGENT_API_URL/api/tasks/<task-id>/dead-letter"
```

For requeue:

```bash
curl -X POST -H "Authorization: Bearer $AGENT_API_TOKEN" \
  -d '{"reset_branch": true}' \
  "$AGENT_API_URL/api/tasks/<task-id>/requeue"
git push origin --delete feature/<task-id>
```

## Output

```
SWEEP COMPLETE — <project>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Orphans found:    <N>
Merged:           <N>  (✓ <N> fast-forward, ✓ <N> via PR)
Dead-lettered:    <N>
Requeued:         <N>
Skipped:          <N>
Failed actions:   <N>  ←  inspect these
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Notes

- **Never bulk-merge without per-orphan review.** A task that "appears mergeable" by tree but breaks the build under integration is worse than no orphan at all.
- For `superseded` orphans, the killer signal is when `git log --grep=<feature keyword>` on main shows recent commits — someone else already did this work.
- The `partial` class is the trickiest. The task says "done" but the branch is incomplete. Read the task's review verdict (`/agent-inspect <task-id>`) to see why review passed.
- If the agent has multiple parallel projects, run this skill per-project. The cross-project version is `/agent-sweep-all` (coming as a separate skill if your operator workflow needs it).
