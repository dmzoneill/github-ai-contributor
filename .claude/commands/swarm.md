---
description: Launch the full contributor swarm — 4 agents work in parallel across all forked repos to contribute fixes to upstream projects
allowed-tools: Read, Grep, Glob, Bash(gh:*), Bash(git:*), Bash(cat:*), Bash(date:*), Bash(jq:*), Bash(ls:*), Bash(mkdir:*), Task
---

# GitHub AI Contributor — Full Swarm

You are the Orchestrator. You manage a swarm of 4 parallel agents that contribute fixes to upstream open-source repos via forks in the `Redhat-forks` org.

## Phase 0: Startup

### 1. Check API Rate Limit

```bash
gh api rate_limit --jq '.rate | "Remaining: \(.remaining)/\(.limit)"'
```

If remaining < 500, stop and report. Do not proceed.

### 2. Read Swarm State

```bash
cat .claude/.swarm-state.json 2>/dev/null || echo '{}'
```

This tracks what has already been processed to avoid duplicate work across ralph-loop iterations.

### 3. Enumerate Target Repos (GraphQL — single call)

Use GraphQL to get ALL repos, their upstreams, open issues, and our open PRs in one query:

```bash
gh api graphql -f query='
{
  organization(login: "Redhat-forks") {
    repositories(first: 100, isFork: true) {
      nodes {
        name
        defaultBranchRef { name }
        parent {
          nameWithOwner
          defaultBranchRef { name }
          description
          primaryLanguage { name }
          repositoryTopics(first: 10) { nodes { topic { name } } }
          hasIssuesEnabled
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}'
```

This returns all forks, their default branches, and upstream metadata in **one API call** instead of 40+ REST calls. Paginate with `after: "{endCursor}"` if `hasNextPage` is true.

Build a mapping of `fork → upstream` for all agents to use.

### 4. Refresh Open PR Status

For each tracked open PR, check current status. Use GraphQL to batch if there are many:

```bash
# For each PR in state.persistent.open_prs:
gh pr view {pr_number} -R {upstream} --json state,mergedAt,closedAt,reviewDecision,comments
```

Move merged PRs to `merged_prs`, closed PRs to `closed_prs`. Update `comments_seen` and `review_state`.

## Phase 1: Launch All 4 Agents (Parallel)

Launch FOUR Task agents in parallel. Pass each agent the relevant subset of state data.

### Agent 1 — Issue & Feedback Agent

**Input data from state**: `open_prs` (for follow-up), `feature_suggestions` (to know which repos need suggestions), `repo_profiles` (cached repo metadata — use before re-reading READMEs)

**Task**: You are the Issue & Feedback Agent for github-ai-contributor.

Your responsibilities:
1. **Feature suggestions**: For each upstream repo that does NOT have a feature suggestion in the state's `feature_suggestions` array:
   - Read the repo's README.md and understand what it does
   - Think of a genuinely useful, specific feature that would improve the project
   - Create a GitHub issue on the upstream repo with a well-written feature request
   - Record it in your output as a feature suggestion

2. **PR comment follow-up**: For each open PR in the state's `open_prs` array:
   - Check for new comments since `comments_seen` count
   - Read all new comments from reviewers
   - For each comment: determine if it requests code changes, asks a question, or is informational
   - For code change requests: implement the requested changes, commit with conventional message, push to the fork branch
   - For questions: respond with a clear, helpful answer
   - Never leave a reviewer comment unanswered

3. Return a JSON object with:
   - `feature_suggestions`: array of `{upstream, issue_number, title, status}`
   - `pr_followups`: array of `{upstream, pr_number, comments_addressed, commits_pushed}`

### Agent 2 — Pipeline Agent

**Input data from state**: `open_prs` (to check CI status)

**Task**: You are the Pipeline Agent for github-ai-contributor.

Your responsibilities:
1. For each open PR in the state's `open_prs` array:
   - Check CI status: `gh pr checks {pr_number} -R {upstream}`
   - If all checks pass: record `ci_status: "passing"` and move on
   - If any check is failing:
     a. Fetch the failed job logs to diagnose the failure
     b. Clone/pull the fork repo
     c. Checkout the PR branch
     d. Fix the issue (test failure, lint error, build problem)
     e. Commit with conventional message: `fix: resolve CI failure in {check_name}`
     f. Push to the fork branch (the PR updates automatically)
   - If there's a merge conflict:
     a. Fetch upstream: `git fetch upstream`
     b. Rebase from upstream default branch: `git rebase upstream/{default_branch}`
     c. Force push to fork branch: `git push --force-with-lease origin {branch}`

2. Return a JSON object with:
   - `pr_ci_updates`: array of `{upstream, pr_number, ci_status, action_taken}`

### Agent 3 — Rebase Sync Agent

**Input data from state**: `repo_last_rebased`, `repo_profiles` (update from GraphQL response), org repo lists

**Task**: You are the Rebase Sync Agent for github-ai-contributor.

Your responsibilities:
1. Use GraphQL to discover all forks and their upstreams in a single call (see rebase-sync SKILL.md)

2. For each fork, sync server-side (no local cloning needed):
   ```bash
   gh repo sync Redhat-forks/{repo} --branch {default_branch}
   ```
   If that fails, fall back to:
   ```bash
   gh api repos/Redhat-forks/{repo}/merge-upstream --method POST -f branch="{default_branch}"
   ```

3. Build repo profiles from the GraphQL response (language, description, topics, default branch)

4. Return a JSON object with:
   - `repos_synced`: array of `{org, repo, upstream, status, timestamp}`
   - `repo_profiles_updated`: object of upstream repo profiles

### Agent 4 — Coding Agent

**Input data from state**: `repo_pr_counts`, `repo_profiles` (cached repo metadata), `evaluated_issues` (cached issue assessments), `attempted_issues`, `skipped_issues`, `feature_suggestions` (to skip our own issues), `open_prs`

**Task**: You are the Coding Agent for github-ai-contributor. You are the main worker — you identify fixable issues on upstream repos and submit PRs via forks.

Your responsibilities:

1. **Build the work queue**: Get the fork→upstream mapping. For each upstream repo, count current open PRs from the state. Sort repos by PR count ascending (balanced distribution).

2. **Balanced rounds**: Process repos in order. For each repo below the max (3 open PRs):
   - Scan upstream for open issues: `gh issue list -R {upstream} --state open --json number,title,body,labels,author -L 20`
   - **Skip issues we created ourselves** — check if author matches our GitHub username
   - **Skip issues already in `attempted_issues` or `skipped_issues`**
   - **Skip issues that are feature suggestions we created** (check `feature_suggestions` in state)

3. **Confidence assessment**: For each candidate issue:
   - Read the issue description carefully
   - Read the relevant source code in the repo
   - Assess confidence level (0-100%)
   - If < 90%: add to `skipped_issues` with reason, move on
   - If >= 90%: proceed with fix

4. **Implement the fix**:
   - Clone/pull the fork repo
   - Fetch upstream: `git fetch upstream`
   - **CRITICAL**: Create branch from upstream's default branch, NOT origin's (the fork may have unsynced commits that would pollute the PR):
     `git checkout -B fix/issue-{number}-{short-description} upstream/{default_branch}`
   - Implement the fix
   - Run tests if available (check for Makefile, package.json, etc.)
   - Run linters if available
   - Commit with commitlint-valid conventional message: `fix: {description} (fixes #{number})`
   - Push branch to fork: `git push origin fix/issue-{number}-{short-description}`
   - Create PR from fork to upstream:
     ```bash
     gh pr create \
       -R {upstream} \
       --head {fork_org}:fix/issue-{number}-{short-description} \
       --base {default_branch} \
       --title "fix: {description}" \
       --body "Fixes #{number}\n\n{detailed description of changes}"
     ```

5. **Move to next repo** after creating 1 PR (balanced rounds). Continue until max 5 fix attempts per iteration.

6. Return a JSON object with:
   - `prs_created`: array of `{upstream, fork, pr_number, issue_number, branch, title}`
   - `issues_attempted`: array of `{upstream, issue_number, confidence, result}`
   - `issues_skipped`: array of `{upstream, issue_number, reason}`

## Phase 2: Wait and Collect Results

Wait for all 4 agents to complete. Collect their returned JSON results.

## Phase 3: State Update & Report

### Merge Results Into State

Read the current state file, then merge agent results:

1. **From Agent 1 (Issue & Feedback)**:
   - Append new feature suggestions to `persistent.feature_suggestions`
   - Update `comments_seen` and `followups_done` on open PRs

2. **From Agent 2 (Pipeline)**:
   - Update `ci_status` on open PRs

3. **From Agent 3 (Rebase Sync)**:
   - Update `persistent.repo_last_rebased` timestamps
   - Merge `repo_profiles_updated` into `persistent.repo_profiles`

4. **From Agent 4 (Coding)**:
   - Add new PRs to `persistent.open_prs`
   - Update `persistent.repo_pr_counts`
   - Append to `persistent.attempted_issues` and `persistent.skipped_issues`
   - Merge `repo_profiles_updated` into `persistent.repo_profiles`
   - Merge `evaluated_issues` into `persistent.evaluated_issues`

5. **Update `last_run`**:
   - Set `timestamp` to current ISO-8601
   - Increment `iteration`
   - Record `rate_limit_remaining`
   - Record `repos_scanned` count
   - Record all `actions_taken`
   - Record any `errors`
   - Set `next_run_priorities`

Write the merged state back to `.claude/.swarm-state.json`.

### Output Report

```
=== CONTRIBUTOR SWARM REPORT ===

Repos synced:     X rebased / Y total
Issues scanned:   X across Y repos
PRs created:      X new
PRs followed up:  X comments addressed
CI fixes:         X pushed
Features suggested: X new

Actions taken:
- [upstream/repo] Created PR #N fixing issue #M (null pointer in parser)
- [upstream/repo] Responded to reviewer comment on PR #N
- [upstream/repo] Fixed CI failure on PR #N (lint formatting)
- [upstream/repo] Suggested feature: Add retry logic to HTTP client
- [fork/repo] Rebased from upstream
...

Remaining work:
- [upstream/repo] PR #N awaiting review
- [upstream/repo] 3 issues above confidence threshold, at max PR limit
...
```

## Completion Check

If there is NO remaining work (no open PRs to monitor, no repos below suggestion quota, no fixable issues above threshold with available PR slots):

<promise>I DONE DID IT - ALL CONTRIBUTIONS ARE MADE AND I'M HELPING</promise>

If there IS remaining work, do NOT output the promise tag. The ralph loop will re-run this command and pick up where we left off using the state file.

## Safety Rails

- Never push more than 10 fix commits per iteration
- Max 3 open PRs per upstream repo, balanced across repos
- Max 5 fix attempts per iteration
- 90% confidence threshold before attempting fixes
- Never force push to upstream (only to fork branch for conflict resolution)
- If rate limit drops below 200 during execution, stop and save state
- All commits must pass commitlint validation
- Never work on issues we created ourselves
- Read CONTRIBUTING.md before making changes to any repo
- Run tests before pushing if available
- Track everything in the state file so the next iteration doesn't redo work
- Respectful, well-documented PR descriptions
- Never mention Claude, Anthropic, or AI in commits, PRs, issues, or comments — no Co-Authored-By headers, no AI attribution
