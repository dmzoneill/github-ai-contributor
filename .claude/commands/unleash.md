---
description: Start the ralph loop contributor swarm — continuously contribute to upstream repos until all work is done
allowed-tools: Read, Bash(gh:*), Bash(cat:*), Bash(date:*), Skill
---

# Unleash the Contributor Swarm

Start a ralph-loop that continuously runs the full contributor swarm until all repos have been contributed to.

## What This Does

Invokes `/ralph-loop` with the `/swarm` command as the prompt. The ralph loop will:
1. Run `/swarm` — which launches 5 parallel agents (issue-feedback, pipeline-fix, rebase-sync, coding-fix, bug-scanner)
2. When `/swarm` finishes an iteration, ralph-loop feeds it back in
3. Each iteration sees previous work via the state file and git history
4. When everything is done (no fixable issues, no unmonitored PRs, all features suggested), the swarm outputs its completion promise and stops

## Launch

Invoke the ralph-loop skill with the swarm as the task:

```
/ralph-loop "Run /swarm to contribute to all upstream repos via forks in Redhat-forks org. Scan upstream issues, assess confidence, implement fixes, create PRs, follow up on reviews, fix CI failures, suggest features. Each iteration: scan for new work, process it, update state file. Output <promise>I DONE DID IT - ALL CONTRIBUTIONS ARE MADE AND I'M HELPING</promise> when zero work remains." --completion-promise "I DONE DID IT" --max-iterations 40
```

## Monitoring

While the swarm runs, check progress:
```bash
cat .claude/.swarm-state.json 2>/dev/null | python3 -m json.tool
```

## Emergency Stop

If things go sideways:
```
/cancel-ralph
```
