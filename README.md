# github-ai-contributor

Prompt-driven, headless AI system that autonomously contributes to open-source repositories. It monitors repos forked into the [`Redhat-forks`](https://github.com/Redhat-forks) GitHub org, scans their upstream repos for open issues, assesses whether it can fix them with >= 90% confidence, and submits PRs back to the upstream. It also suggests features.

Runs every 3 hours via GitHub Actions (1-hour timeout per run), fully headless using `claude -p`. Contains **no application code** — entirely orchestrated by [Claude Code](https://claude.ai/code) via markdown prompts.

## How It Works

```mermaid
flowchart TD
    U["Upstream Repo<br/>(someone else's project)"] -.->|fork already exists| F["Our Fork<br/>Redhat-forks"]
    F --> S["Sync fork from upstream<br/>(gh repo sync)"]
    S --> SC["Scan upstream for open issues"]
    SC --> AS{"Confidence<br/>>= 90%?"}
    AS -->|no| SK["Skip issue"]
    AS -->|yes| BR["Create fix branch<br/>from upstream/main"]
    BR --> FX["Implement fix + run tests"]
    FX --> PS["Push branch to fork"]
    PS --> PR["Create PR:<br/>fork branch → upstream"]
    PR --> MN["Monitor PR lifetime"]
    MN --> RV["Respond to reviews"]
    MN --> CI["Fix CI failures"]
    MN --> RB["Rebase on conflicts"]
    MN --> DN{"Merged or<br/>closed?"}
    DN -->|yes| DONE["Done — stop tracking"]
    DN -->|no| MN
    style U fill:#30363d,color:#fff
    style F fill:#1f6feb,color:#fff
    style PR fill:#238636,color:#fff
    style DONE fill:#238636,color:#fff
    style SK fill:#da3633,color:#fff
```

## Architecture

Four specialized agents run in parallel, coordinated by an orchestrator:

```mermaid
graph TD
    O[Orchestrator<br/>/swarm] --> A1[Agent 1<br/>Issue & Feedback]
    O --> A2[Agent 2<br/>Pipeline Fix]
    O --> A3[Agent 3<br/>Rebase Sync]
    O --> A4[Agent 4<br/>Coding Fix]
    A1 --> GH[GitHub API<br/>Upstream Repos]
    A2 --> GH
    A3 --> GH
    A4 --> GH
    A4 --> FK[Fork Repos<br/>Redhat-forks]
    A3 --> FK
    style O fill:#1f6feb,color:#fff
    style A1 fill:#238636,color:#fff
    style A2 fill:#238636,color:#fff
    style A3 fill:#238636,color:#fff
    style A4 fill:#da3633,color:#fff
    style GH fill:#30363d,color:#fff
    style FK fill:#30363d,color:#fff
```

| Agent | Role |
|---|---|
| **Issue & Feedback** | Suggests features on upstream repos. Follows up on reviewer comments on our open PRs. |
| **Pipeline Fix** | Monitors CI status on our PRs. Diagnoses and fixes failures. Handles merge conflicts. |
| **Rebase Sync** | Keeps all forks synced with upstream via rebase. |
| **Coding Fix** | The main worker. Scans issues, assesses confidence, implements fixes, creates PRs. |

## Orchestration Flow

```mermaid
graph TD
    S["Phase 0: Startup<br/>Rate limit check + read state"] --> P1["Phase 1: Launch 4 agents<br/>(parallel)"]
    P1 --> A1["Agent 1: Issue & Feedback"]
    P1 --> A2["Agent 2: Pipeline Fix"]
    P1 --> A3["Agent 3: Rebase Sync"]
    P1 --> A4["Agent 4: Coding Fix"]
    A1 --> P2["Phase 2: Wait + collect results"]
    A2 --> P2
    A3 --> P2
    A4 --> P2
    P2 --> P3["Phase 3: Merge state + report"]
    P3 --> CHK{"Remaining<br/>work?"}
    CHK -->|yes| RL["Ralph loop restarts"]
    CHK -->|no| DONE["Done"]
    RL --> S
    style S fill:#30363d,color:#fff
    style DONE fill:#238636,color:#fff
    style A4 fill:#da3633,color:#fff
```

## Relationship to github-ai-maintainer

[`github-ai-maintainer`](https://github.com/dmzoneill/github-ai-maintainer) works on repos we **own** (dmzoneill's repos). This project works on **other people's repos** via forks. Same prompt-driven architecture, different target.

## Safety Rails

- **90% confidence threshold** before attempting any fix
- **Max 3 open PRs per upstream repo**, balanced across repos
- **Max 5 fix attempts per iteration**, max 20 iterations per run
- **Rate limit monitoring** — won't start if < 500 remaining, stops if < 200
- **Commitlint validation** on all commits (conventional commit format)
- **Never force push to upstream** — only to fork branches for conflict resolution
- **Runs tests before pushing** when available
- **Reads CONTRIBUTING.md** and upstream conventions before contributing
- **Full PR ownership** — monitors, responds to reviews, fixes CI, rebases on conflicts
- **Never works on self-created issues** — feature suggestions are for the community

## Project Structure

```
github-ai-contributor/
├── CLAUDE.md                     # Master specification for Claude Code
├── version                       # v0.0.1
├── LICENSE                       # Apache 2.0
├── .claude/
│   ├── .swarm-state.json         # Persistent state across runs
│   ├── commands/
│   │   ├── swarm.md              # Main orchestration (4 parallel agents)
│   │   └── unleash.md            # Ralph-loop launcher
│   └── skills/
│       ├── issue-feedback/       # Agent 1: features + PR follow-up
│       ├── pipeline-fix/         # Agent 2: CI failure diagnosis & fix
│       ├── rebase-sync/          # Agent 3: fork-to-upstream sync
│       └── coding-fix/           # Agent 4: issue fixing + PR creation
├── .github/workflows/
│   ├── contribute.yml            # Every 3 hours via cron
│   └── main.yml                  # CI/CD dispatch wrapper
└── .gitignore
```

## Running

### Automated (GitHub Actions)

The `contribute.yml` workflow runs every 3 hours with a 1-hour timeout. It can also be triggered manually via `workflow_dispatch`.

### Manual (Claude Code CLI)

```bash
# Run the full swarm once
claude -p "/swarm"

# Run continuously until all work is done
claude -p "/unleash"
```

## Environment Variables

| Variable | Purpose |
|---|---|
| `GITHUB_TOKEN` / `PROFILE_HOOK` | GitHub API access |
| `CLAUDE_CODE_USE_VERTEX` | Enable Vertex AI backend |
| `ANTHROPIC_VERTEX_PROJECT_ID` | GCP project for Vertex |
| `CLOUD_ML_REGION` | GCP region |
| `GOOGLE_APPLICATION_CREDENTIALS` | Path to GCP credentials JSON |

## License

Apache 2.0 - see [LICENSE](LICENSE).
