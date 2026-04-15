# Architecture

A walkthrough of how the autonomous dev squad coordinates.

## The big picture

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

## The actors

### 1. PM agent (`agents/dev/pm-agent.md`)

A teammate that runs in the background. Its entire job is board mechanics:

- On startup, it scans the project board via `gh project item-list`
- For each Backlog ticket, it fetches the issue body and parses a `depends-on: #1, #2, #3` line (plain text, case-insensitive)
- It builds a set of "done" tickets (Done + Review both count) and marks any Backlog ticket whose dependencies are all in that set as **unblocked**
- It promotes up to 1 unblocked frontend and 1 unblocked backend ticket to Ready — never more, to avoid flooding
- It sends a message to dev-lead with the result: `READY_TICKETS_AVAILABLE`, `WAITING`, `PROJECT_COMPLETE`, or `ALL_BLOCKED`
- It goes idle and waits for dev-lead to say "refill" after a cycle finishes

The PM never spawns agents, never writes code, never moves tickets forward beyond Ready. It's a pure board-state machine.

### 2. Dev-lead (`agents/dev/team-lead-agent.md`)

The orchestrator. Runs as a teammate but keeps a **single continuous turn** from the moment PM says "Ready tickets available" until PM says "stop" — it never goes idle between cycles. This is critical: if dev-lead ends its turn, the loop dies.

Per cycle:

1. Receive Ready tickets from PM, move them Ready → In Progress
2. Read the appropriate prompt file (`frontend.md`, `backend.md`) and spawn sub-agents **in parallel** with `isolation: "worktree"`
3. When sub-agents return PR links, rebase each PR on `origin/main`
4. Spawn the test sub-agent with all PRs, wait for results
5. (Optional) Invoke a code-review skill for audit trail
6. For any frontend PR that passed: spawn the QA sub-agent, wait for results
7. Move passing tickets to Review, increment retry counts on failures
8. SendMessage to PM: "refill"
9. Parse PM's reply — if `READY_TICKETS_AVAILABLE`, loop back to step 1; otherwise produce the final report

Dev-lead handles retry logic (max 3 attempts per ticket for test or QA failures), stale-progress detection (2 consecutive empty cycles → stop), and the frontend QA gate (a frontend ticket cannot reach Review without a QA result).

### 3. Sub-agents (the workers)

Each sub-agent is spawned fresh via the `Agent` tool, does one thing, and dies. They don't know about the loop, the PM, or each other. They just take inputs and return a result string.

- **`backend.md`** — assigns the issue, implements code in its worktree, runs ticket-specific pytest files, opens a PR, returns the link
- **`frontend.md`** — same pattern, but for frontend tickets; optionally invokes the `frontend-design` skill for design-heavy tickets
- **`test-agent.md`** — per PR: classifies scope, runs pytest / jest / tsc / lint / build, posts a markdown report to the PR and the issue
- **`qa-agent.md`** — per frontend PR: applies the diff on a temp branch, boots dev servers, uses Playwright MCP tools to test acceptance criteria, posts results, and **always** runs cleanup (revert to main, restart servers)

All sub-agents run with `mode: "bypassPermissions"` because they need to execute `git`, `gh`, `npm`, `pytest`, etc. without prompting.

## Why three dev-team skills?

Claude Code's teams API has a known issue where teammates spawned as `"general-purpose"` don't always receive the `Agent` tool at runtime — which means dev-lead can't spawn sub-agents. The three skills are three workarounds:

| Skill                | PM runs as... | Dev-lead runs as... | Sub-agents spawned by... |
|----------------------|---------------|---------------------|--------------------------|
| `/dev-team`          | teammate      | teammate            | dev-lead (if Agent tool is present) |
| `/dev-team-hybrid`   | teammate      | **main thread**     | main thread              |
| `/dev-team-proxy`    | teammate      | teammate            | main thread (on spawn requests from dev-lead) |

Use `/dev-team` first. If dev-lead hits `FATAL: Agent tool missing`, fall back to `/dev-team-hybrid` — it's the most reliable. `/dev-team-proxy` is a middle ground that keeps dev-lead as a teammate but relies on the main thread as a spawn relay.

## The QA track (`/qa`, `/qa-fix`)

Separate from dev-team. The QA flow tests the integrated app **after** dev cycles have merged and uses a completely different loop:

1. `run-qa.md` orchestrator reads `qa-state.md`, picks the next module (one module per run), and kicks off cycle 1
2. `system-qa.md` agent runs Playwright browser tests against that module, writes results to `qa-reports/{module}-bugs.md`
3. If bugs are found, `fullstack.md` agent reads the bug report, fixes the bugs across frontend + backend, pushes to a fix branch, and opens/updates a PR
4. User restarts dev servers on the fix branch; orchestrator runs cycle 2; max 3 cycles per module, then STOP

This track is intentionally lower-automation than the dev-team loop — the user stays in the loop to restart servers between fix cycles because QA often involves state that doesn't reset cleanly.

`/qa-fix` is a one-shot variant for free-text bug reports ("the persona chips aren't highlighting on click") — it skips the state file and just runs one test-fix-retest cycle autonomously.

## Message protocol

PM ↔ dev-lead communicate via `SendMessage`. The signals are plain strings in the message body, and parsers look for exact substrings:

- `READY_TICKETS_AVAILABLE` — PM found work, dev-lead should proceed
- `WAITING` — Backlog has tickets but they're all blocked by in-progress work
- `PROJECT_COMPLETE` — board is empty (all Done or Review)
- `ALL_BLOCKED` — nothing unblocked, nothing in progress, stuck

When dev-lead sees any of `WAITING`, `PROJECT_COMPLETE`, or `ALL_BLOCKED`, it must output its final report **in the same turn** and stop — no more messages to PM.

The `team-lead-proxy.md` variant adds a second protocol for dev-lead ↔ main-thread spawn relay:

- `SPAWN_REQUEST` (dev-lead → main) with type, issue, PR, mode, and additional context
- `SPAWN_RESULT` (main → dev-lead) with status and the sub-agent's full return message
- `REBASE_REQUEST` / `REBASE_RESULT` for rebase operations that the proxy handles
- Multiple requests in one message separated by `===` for parallel spawns

## Worktree isolation

Dev sub-agents are spawned with `isolation: "worktree"`, which gives each one its own checked-out copy of the repo in a separate directory. This means:

- Two parallel dev agents can modify the same file without conflict — git handles merging at PR time
- A failed dev agent's half-written code doesn't pollute the main checkout
- Cleanup is automatic when the agent terminates (if nothing changed) or explicit otherwise

## Failure modes worth knowing

- **Rebase conflicts** after a dev sub-agent finishes → dev-lead spawns a new dev sub-agent in "fix mode" with the conflict details
- **Test failure on a PR** → increment retry count, spawn dev in fix mode, re-run test agent (max 3 attempts)
- **QA `BLOCKED` result** (infra issue, not a real bug) → do **not** count as a retry, move the ticket to Review anyway
- **2 consecutive empty cycles** (zero tickets reached Review) → dev-lead stops with "stuck" reason and produces a final report
- **Teammate-Agent-tool bug** → fall back to a hybrid/proxy skill (see table above)
