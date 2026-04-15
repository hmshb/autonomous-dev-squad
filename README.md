# autonomous-dev-squad

> An autonomous dev team for [Claude Code](https://claude.com/claude-code) that ships GitHub tickets end-to-end — PM, dev, test, and QA agents that coordinate through your GitHub Projects v2 board with zero human babysitting.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-8a2be2)](https://claude.com/claude-code)

---

## What is this?

A production-tested set of **Claude Code agents and skills** that work together as an autonomous development team. You drop a ticket on your GitHub project board, type `/dev-team`, and a PM agent promotes it to Ready, a dev-lead spawns backend/frontend sub-agents to implement it, a test agent validates the PR, a QA agent runs Playwright browser tests, and the ticket lands in Review — all in one continuous run.

Built as a real multi-agent system, not a demo: the agents communicate via Claude Code's experimental Teams API, respect a 1-FE-+-1-BE Ready slot limit, parse `depends-on:` lines out of issue bodies, run dev work in isolated git worktrees, retry failures up to 3 times, and produce a final report when the board is drained.

## The flow

```
You: "/dev-team"
  │
  ▼
1. PM agent   — scans the board, parses depends-on: lines, promotes
                the next unblocked FE + BE tickets from Backlog to Ready
  │
  ▼
2. Dev-lead   — picks them up, moves to In Progress, spawns dev sub-agents
                in parallel worktrees (one per ticket)
  │
  ▼
3. Dev agents — implement the ticket, run TDD tests, open a PR, return the link
  │
  ▼
4. Test agent — runs pytest/jest/tsc/lint/build on each PR, posts results
  │
  ▼
5. QA agent   — for frontend PRs only: boots dev servers, runs Playwright
                browser tests against the acceptance criteria, posts results
  │
  ▼
6. Dev-lead   — moves passing tickets to Review, calls PM for refill
  │                                                   │
  └────────────── loop until empty ───────────────────┘
  │
  ▼
7. Final report — cycles completed, tickets shipped, anything flagged
```

## What's inside

### Agents (9)

| Agent | Role |
|---|---|
| `agents/dev/pm-agent.md` | Reads the board, resolves `depends-on:` dependencies, promotes Backlog → Ready with a 1 FE + 1 BE slot limit |
| `agents/dev/team-lead-agent.md` | Orchestrator teammate that spawns dev/test/QA sub-agents in a continuous loop |
| `agents/dev/backend.md` | Implements one backend ticket, runs TDD tests, opens a PR |
| `agents/dev/frontend.md` | Implements one frontend ticket; optionally invokes `frontend-design` for design-heavy work |
| `agents/dev/test-agent.md` | Validates PRs (pytest / jest / tsc / lint / build), posts reports on PR + issue |
| `agents/dev/qa-agent.md` | Applies the PR diff, boots servers, runs Playwright browser tests against acceptance criteria |
| `agents/qa/run-qa.md` | Module-by-module QA orchestrator with 3-cycle test/fix/re-test loop |
| `agents/qa/system-qa.md` | Per-module Playwright browser testing with bug report output |
| `agents/qa/fullstack.md` | Reads QA bug reports, fixes bugs across frontend + backend, opens a fix PR |

### Skills (5)

| Skill | What it does |
|---|---|
| `skills/create-ticket` | `/create-ticket` — create or update GitHub issues with labels and project-board placement in one command |
| `skills/dev-team` | `/dev-team` — full autonomous mode (PM + dev-lead as teammates) |
| `skills/dev-team-hybrid` | `/dev-team-hybrid` — PM as teammate, main thread as dev-lead (avoids the teammate Agent-tool bug) |
| `skills/qa` | `/qa` — module-by-module QA with auto fix-and-retest loop |
| `skills/qa-fix` | `/qa-fix` — instruction-driven QA ("the chips aren't highlighting on click" → test + fix + PR) |

## Requirements

- [Claude Code](https://claude.com/claude-code) installed
- [`gh` CLI](https://cli.github.com/) authenticated (`gh auth login`)
- A GitHub repository with a **Projects v2 board** containing a single-select `Status` field with the options: `Backlog`, `Ready`, `In Progress`, `Review`, `Done`
- Labels `frontend` and `backend` on the repo
- **Optional but recommended:** [Playwright MCP](https://github.com/microsoft/playwright-mcp) for the QA agents
- Uses Claude Code's experimental teams flag: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (see `examples/settings.json`)

## Install

### Option A — Project-level (recommended)

```bash
git clone https://github.com/hmshb/autonomous-dev-squad.git
cd your-project
cp -r ../autonomous-dev-squad/agents  .claude/
cp -r ../autonomous-dev-squad/skills  .claude/
```

### Option B — User-level (available in every project)

```bash
git clone https://github.com/hmshb/autonomous-dev-squad.git
cp -r autonomous-dev-squad/agents  ~/.claude/
cp -r autonomous-dev-squad/skills  ~/.claude/
```

Then follow **[docs/setup.md](docs/setup.md)** to substitute the GitHub project placeholders with your actual project IDs (5-minute setup using `gh` CLI).

## Quick start

Once installed and configured:

```
/dev-team
```

That's it. The PM agent will scan your project board, move the next unblocked ticket(s) to Ready, and hand off to dev-lead. Dev-lead runs the cycle loop until the board is drained (or flagged tickets need human review), and produces a final report.

For one-off ticket creation:
```
/create-ticket build a login page with email + password --scope frontend
```

For QA-driven testing:
```
/qa                                  # walk through all modules
/qa-fix "the persona dropdown truncates long names"
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                            Main thread                               │
│                         (user / orchestrator)                        │
└────────┬───────────────────────────────────────┬─────────────────────┘
         │                                       │
         │ spawn teammate                        │ spawn teammate
         │ (background)                          │ (continuous turn)
         ▼                                       ▼
  ┌────────────────┐                   ┌────────────────────┐
  │   PM agent     │◄── SendMessage ───┤   Dev-lead         │
  │                │    ("refill")     │   (team-lead-agent)│
  │ scans board    │                   │                    │
  │ reads          │── "READY: #28" ──►│ picks tickets off  │
  │ depends-on:    │                   │ Ready, spawns      │
  │ promotes       │                   │ sub-agents in      │
  │ Backlog→Ready  │                   │ parallel worktrees │
  └────────────────┘                   └──┬──────┬──────┬───┘
                                          │      │      │
                                          │      │      │  spawn
                                          ▼      ▼      ▼
                                  ┌──────────┐ ┌──────┐ ┌──────┐
                                  │ backend  │ │ front│ │ test │
                                  │ sub-agent│ │ end  │ │ agent│
                                  │          │ │ sub  │ │      │
                                  │ worktree │ │ wt   │ │ tsc/ │
                                  │ pytest   │ │ jest │ │ lint │
                                  │ gh pr    │ │ gh pr│ │ build│
                                  └──────────┘ └──────┘ └──────┘
                                                            │
                                                            │ on PASS
                                                            ▼
                                                     ┌──────────┐
                                                     │ qa agent │
                                                     │ Playwright│
                                                     │ MCP      │
                                                     └──────────┘
```

### How the actors coordinate

**PM agent** runs in the background as a teammate. Its whole job is board mechanics: it reads each Backlog issue's body, parses `depends-on: #1, #2` lines, and promotes up to 1 frontend + 1 backend unblocked ticket to Ready. Then it sends dev-lead a signal (`READY_TICKETS_AVAILABLE`, `WAITING`, `PROJECT_COMPLETE`, or `ALL_BLOCKED`) and goes idle until dev-lead says "refill."

**Dev-lead** is the orchestrator. It runs as a teammate but keeps a **single continuous turn** from "Ready tickets available" until PM says stop — it never goes idle between cycles, because that would kill the loop. Per cycle:

1. Receive Ready tickets from PM, move them Ready → In Progress
2. Read the agent prompt files (`frontend.md`, `backend.md`) and spawn dev sub-agents **in parallel** with `isolation: "worktree"` (each sub-agent gets its own checked-out copy of the repo)
3. When sub-agents return PR links, rebase each PR on `origin/main`
4. Spawn the test sub-agent with all PRs, wait for results
5. For any frontend PR that passed: spawn the QA sub-agent (Playwright MCP), wait for results
6. Move passing tickets to Review, increment retry counts on failures (max 3 per ticket)
7. SendMessage to PM: "refill" — and loop

**Sub-agents** (backend, frontend, test, qa) are spawned fresh via the `Agent` tool, do one thing, and die. They don't know about the loop, the PM, or each other. They take inputs and return a result string. Dev sub-agents run in isolated git worktrees so two parallel agents can touch the same file without conflicts.

### Why two dev-team skills?

Claude Code's teams API has a known issue where teammates spawned as `"general-purpose"` don't always receive the `Agent` tool at runtime — which means dev-lead can't spawn sub-agents. The two skills give you a primary path and a reliable fallback:

| Skill                | PM runs as... | Dev-lead runs as... | Sub-agents spawned by...                 |
|----------------------|---------------|---------------------|------------------------------------------|
| `/dev-team`          | teammate      | teammate            | dev-lead (if Agent tool is present)      |
| `/dev-team-hybrid`   | teammate      | **main thread**     | main thread                              |

Use `/dev-team` first. If dev-lead hits `FATAL: Agent tool missing`, fall back to `/dev-team-hybrid` — it's simpler and more reliable.

### Message protocol

PM ↔ dev-lead communicate via `SendMessage`. The signals are plain strings in the message body; parsers look for exact substrings:

- `READY_TICKETS_AVAILABLE` — PM found work, dev-lead should proceed
- `WAITING` — Backlog has tickets but they're all blocked by in-progress work
- `PROJECT_COMPLETE` — board is empty (all Done or Review)
- `ALL_BLOCKED` — nothing unblocked, nothing in progress, stuck

When dev-lead sees any of `WAITING`, `PROJECT_COMPLETE`, or `ALL_BLOCKED`, it must output its final report **in the same turn** and stop — no more messages to PM.

See **[docs/architecture.md](docs/architecture.md)** for the full walkthrough including the QA track, worktree isolation details, and failure modes.

## Limitations and known issues

- **Teammate Agent tool bug.** In some Claude Code versions, teammates spawned with `subagent_type: "general-purpose"` don't receive Agent tool access at runtime. `/dev-team` is the simplest (use if it works for you); `/dev-team-hybrid` is the fallback that makes the main thread the dev-lead, sidestepping the bug entirely.
- **Module list in QA agents is project-specific.** The default modules (`auth, personas, generation, history, admin, uiux`) are an example set from the project this was extracted from. Edit `agents/qa/system-qa.md` and `agents/qa/run-qa.md` to match your project's modules before running QA.
- **Requires GitHub Projects v2.** The PM agent and create-ticket skill use the newer projects API, not the deprecated v1 boards.
- **Opinionated monorepo layout.** The test and QA agents assume a `backend/` + `frontend/` directory layout (FastAPI + Next.js by default). Adapt the install/start commands for your stack — stack-agnostic sections are marked inline.
- **Experimental teams flag.** Relies on `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, which may change between Claude Code releases.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). PRs welcome — especially for:
- Alternative stack adapters (Go, Ruby, Rust, Django, Rails, etc.)
- Additional QA module templates
- Better teammate-spawn stability fixes
- Documentation improvements

## License

MIT — see [LICENSE](LICENSE).
