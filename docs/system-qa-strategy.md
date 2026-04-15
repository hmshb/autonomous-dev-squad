# System QA + Full-Stack Bug Fix Strategy

> This is a design document explaining how the QA agents work and why. It's kept as a reference for contributors who want to understand or adapt the QA track for their own projects.

## Context

When a set of tickets is merged to main, the individual unit tests may pass while the integrated system still has bugs — wrong wiring between components, regressions in flows that cross module boundaries, state management issues that only show up in a real browser. The QA agents are designed to catch these.

The QA flow runs end-to-end tests against the integrated system using Playwright MCP, fixes bugs found, and verifies fixes — all automated with light orchestration from the main thread.

## Strategy: Two Agents + Light Orchestration

Two agent files handle the work. The main thread orchestrates the QA → fix loop per module. No teams needed.

### 1. `agents/qa/system-qa.md`

A system-level QA agent that tests one module at a time using Playwright MCP.

**Key design:**
- Takes a **module name** and **branch name** as input
- Tests against the specified branch (main for cycle 1, fix branch for subsequent cycles)
- Uses Playwright MCP tools to test all flows in the given module
- Writes results to `qa-reports/{module}-bugs.md` (creates or updates)
- Returns summary: "QA COMPLETE — X/Y passed, Z bugs found"
- External services unavailable → mark BLOCKED (not FAIL)
- Defensive coding issues seen only during infrastructure failures → OBSERVATION (informational, not a bug)

**Bug report format (`qa-reports/{module}-bugs.md`):**
```markdown
# QA Report: {Module Name}
**Branch:** {branch}
**Cycle:** {N}
**Date:** {date}
**Overall:** X/Y PASS

| # | Scenario | Status | Description | Expected | Actual |
|---|----------|--------|-------------|----------|--------|
| 1 | ...      | PASS/FAIL/BLOCKED | ... | ... | ... |

## Open Bugs
- BUG-1: {description} — {status: OPEN/FIXED/BLOCKED}

## Observations
- OBS-1: {defensive coding issue observed during infra failure}
```

### 2. `agents/qa/fullstack.md`

A full-stack fix agent that can edit both frontend and backend code.

**Key design:**
- Takes a **module name** as input
- Reads `qa-reports/{module}-bugs.md` to get all OPEN bugs
- Fixes all bugs for that module in a single pass
- Works on a persistent branch: `fix/{module}-qa` (creates on first run, reuses on subsequent)
- Pushes commits to the same branch/PR across cycles
- Runs both backend tests (`pytest`) and frontend tests (`jest` + `tsc` + build) — adapt these to your stack
- Creates PR on first cycle, pushes additional commits on subsequent cycles
- Does NOT merge — PR stays open until the module passes clean

## Orchestration Loop (Main Thread via `run-qa.md`)

```
For each module (in dependency order):

  1. Create fix branch: fix/{module}-qa (from main)

  For cycle = 1 to 3:
    2. Spawn system-qa agent:
       - Module: {module}
       - Branch: main (cycle 1) or fix/{module}-qa (cycle 2+)
       - Output: qa-reports/{module}-bugs.md

    3. Read bug report
       - If 0 open bugs → module CLEAN, break
       - If bugs found → continue

    4. Spawn fullstack agent:
       - Module: {module}
       - Branch: fix/{module}-qa
       - Reads qa-reports/{module}-bugs.md
       - Fixes all OPEN bugs, pushes to branch
       - Creates/updates PR (does NOT merge)

    5. Continue to next cycle

  After loop:
    - If module clean → merge PR, move to next module
    - If still bugs after 3 cycles → leave PR open, log remaining
      bugs in report for human review, move to next module
```

## Module Testing Order

Test modules in **dependency order** — earlier modules must pass before later ones make sense to test.

Example order from a typical SaaS app (adapt to your project):

| # | Module | Why this order |
|---|--------|----------------|
| 1 | Auth | Foundation — everything needs auth |
| 2 | Core entity CRUD (personas, products, etc.) | Needed by main features |
| 3 | Core feature (generation, checkout, etc.) | Depends on the entities above |
| 4 | History / records | Depends on core feature data |
| 5 | Admin | Needs data from above modules |
| 6 | UI/UX polish | Independent, test last |

## Key Rules

- **No GitHub issues for bugs.** Bugs are tracked in `qa-reports/{module}-bugs.md` — the fix agent reads them directly, skipping the issue-tracker round trip.
- **No merge until clean.** The fix PR stays open until system-qa passes with 0 bugs for that module.
- **Max 3 cycles per module.** Avoid infinite loops; remaining bugs are left in the report for human review.
- **Single PR per module.** All fixes for a module go in one PR — a 4th-cycle fix appends commits to the same PR.
- **Exit early on clean pass.** If 0 bugs on first QA run, skip the fix cycle entirely.
- **Sequential modules.** Don't start the next module until the current one is clean (or max cycles reached) — testing out of order muddies the signal.
- **Observations ≠ bugs.** Error-handling behavior observed only during infrastructure failures is logged as an OBSERVATION and does not trigger fix cycles.

## Why this shape

- **Per-module bug reports** keep the fix agent's context small and focused. A fix agent that sees only its module's bugs is less likely to accidentally fix the wrong thing.
- **State file (`qa-state.md`)** makes the orchestrator resumable across sessions — you can stop and restart without losing track of which modules are done.
- **Module-by-module** instead of all-at-once lets you ship fixes incrementally and catch regressions early.
- **Server restart gate** between cycles is intentional friction — it forces the user to verify the fix branch is actually running before burning more QA tokens.
