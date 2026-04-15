---
name: dev-team-hybrid
description: Hybrid dev team — PM as teammate, main thread as dev-lead orchestrator. Avoids the Agent tool bug in teammate initialization.
---

Run the dev team in hybrid mode: PM runs as an autonomous teammate managing
the board, while the main thread acts as dev-lead (orchestrator), spawning
dev/test/QA sub-agents directly. This avoids the known bug where teammates
spawned with `subagent_type: "general-purpose"` don't receive Agent tool access.

---

## Configuration (REQUIRED — replace before first run)

This skill uses placeholder tokens. See `docs/setup.md` at the repo root
for instructions.

---

## Step 0 — Preflight checks (MANDATORY)

Verify all required agent prompt files exist:
```bash
for f in \
  .claude/agents/dev/pm-agent.md \
  .claude/agents/dev/frontend.md \
  .claude/agents/dev/backend.md \
  .claude/agents/dev/test-agent.md \
  .claude/agents/dev/qa-agent.md; do
  [ -f "$f" ] && echo "OK: $f" || echo "MISSING: $f"
done
```

If ANY file is reported MISSING, stop immediately and output:
```
PREFLIGHT FAILED: The following required files are missing:
  - {list missing files}
Aborting. Fix the missing files and re-run the skill.
```

Do NOT proceed to Step 1 until all files are confirmed present.

---

## Step 1 — Create team and spawn PM

1. Create a team named `"dev-team-hybrid"`.
2. Run `sleep 3` to let the team initialize.
3. Read the full contents of `.claude/agents/dev/pm-agent.md`.
4. Spawn PM as a teammate:
   - name: `"pm"`
   - team_name: `"dev-team-hybrid"`
   - subagent_type: `"general-purpose"`
   - mode: `"bypassPermissions"`
   - run_in_background: `true`
   - prompt: the FULL content of `pm-agent.md`, BUT replace all
     references to `"dev-lead"` with `"team-lead"` in the SendMessage
     targets (so PM messages the main thread instead of a dev-lead
     teammate). Add this instruction at the top of the prompt:
     ```
     IMPORTANT: You are in hybrid mode. Send all your messages to
     "team-lead" (the main thread) instead of "dev-lead".
     Wherever these instructions say SendMessage to "dev-lead",
     send to "team-lead" instead.
     ```
     Then append: `BEGIN NOW: Run startup workflow.`

The main thread does NOT go idle — it acts as dev-lead from here on.

---

## Step 2 — Wait for PM's startup message

Wait for PM to send its initial board analysis with Ready tickets and a signal.

Parse the signal:
- `READY_TICKETS_AVAILABLE` → proceed to Step 3
- `WAITING` → go to Step 7 (final report)
- `PROJECT_COMPLETE` → go to Step 7 (final report)
- `ALL_BLOCKED` → go to Step 7 (final report)

---

## Step 3 — Dev-lead loop: Spawn dev sub-agents

You (the main thread) now act as the team lead orchestrator. Initialize:
- `cycle_count = 1`
- `retry_counts = {}` (map of issue number → retry count)
- `stale_cycles = 0` (consecutive cycles with zero tickets reaching Review)
- `tickets_to_review = []` (cumulative list across all cycles)
- `tickets_flagged = []` (cumulative list across all cycles)
- `cycle_log = []` (for final report)

### 3a — Pull latest main
```bash
git checkout main && git pull origin main
```

### 3b — Read agent prompts
- If there's a frontend ticket → Read `.claude/agents/dev/frontend.md`
- If there's a backend ticket → Read `.claude/agents/dev/backend.md`

### 3c — Move tickets Ready → In Progress
For each ticket being picked up:
1. Find its item ID:
   ```bash
   gh project item-list ${PROJECT_NUMBER} --owner ${OWNER} --format json --limit 100 | python -c "
   import json,sys
   for item in json.load(sys.stdin).get('items',[]):
       if item['content'].get('number') == {ISSUE_NUMBER}:
           print(item['id']); break"
   ```
2. Move to In Progress:
   ```bash
   gh project item-edit --project-id ${PROJECT_ID} \
     --id {ITEM_ID} \
     --field-id ${STATUS_FIELD_ID} \
     --single-select-option-id ${STATUS_IN_PROGRESS}
   ```

### 3d — Spawn dev sub-agents
Log before spawning:
```
[Cycle {N}] SPAWNING frontend-dev for #{ISSUE} — worktree isolation
[Cycle {N}] SPAWNING backend-dev for #{ISSUE} — worktree isolation
```

Spawn dev sub-agents using the `Agent` tool:
- `subagent_type: "general-purpose"`
- `isolation: "worktree"`
- `mode: "bypassPermissions"`
- prompt: FULL content of `frontend.md` or `backend.md` + issue number +
  instruction to implement, test, create PR, and return the PR link

Spawn them **IN PARALLEL** (multiple Agent calls in one message).

When they return, log:
```
[Cycle {N}] RETURNED frontend-dev for #{ISSUE} — {DONE/BLOCKED} — PR #{PR_NUMBER}
[Cycle {N}] RETURNED backend-dev for #{ISSUE} — {DONE/BLOCKED} — PR #{PR_NUMBER}
```

---

## Step 4 — Rebase and test

### 4a — Rebase each PR on main
```
[Cycle {N}] REBASING PR #{PR_NUMBER} on origin/main
```
```bash
gh pr checkout {PR_NUMBER}
git fetch origin main
git rebase origin/main
git push --force-with-lease
```
If rebase fails with conflicts, spawn a new dev sub-agent in fix mode.

### 4b — Spawn test sub-agent
```
[Cycle {N}] SPAWNING test-agent for PRs: #{PR1}, #{PR2}
```

Read `.claude/agents/dev/test-agent.md`. Spawn test sub-agent:
- `subagent_type: "general-purpose"`
- `mode: "bypassPermissions"`
- prompt: FULL content of `test-agent.md` + PR numbers + issue numbers

When it returns, log:
```
[Cycle {N}] RETURNED test-agent — PR #{PR_NUMBER}: {PASS/FAIL}
```

For each PR:
- **PASS + backend**: Mark as ready for Review (Step 5)
- **PASS + frontend**: MUST proceed to QA (Step 4c). Do NOT skip.
- **FAIL**: Increment `retry_counts[issue]`. If under 3: spawn new dev
  sub-agent in fix mode, then re-test. If at 3: flag ticket (see Retry
  limit section below).

### 4c — Spawn QA sub-agent (MANDATORY for frontend PRs)

**GATE CHECK**: Before moving ANY frontend PR to Review, verify QA has run.

```
[Cycle {N}] SPAWNING qa-agent for PR #{PR_NUMBER} (issue #{ISSUE})
```

Read `.claude/agents/dev/qa-agent.md`. Spawn QA sub-agent:
- `subagent_type: "general-purpose"`
- `mode: "bypassPermissions"`
- prompt: FULL content of `qa-agent.md` + issue/PR numbers

When it returns, log:
```
[Cycle {N}] RETURNED qa-agent — PR #{PR_NUMBER}: {PASS/FAIL/BLOCKED}
```

- **PASS**: Move to Review
- **FAIL**: Increment retry count. If under 3: spawn dev fix, re-test,
  re-QA. If at 3: flag ticket.
- **BLOCKED** (infra issue): Do NOT count as retry. Skip QA, move to Review.

---

## Step 5 — Cycle complete

Move all passing tickets In Progress → Review on the project board:
```bash
gh project item-edit --project-id ${PROJECT_ID} \
  --id {ITEM_ID} \
  --field-id ${STATUS_FIELD_ID} \
  --single-select-option-id ${STATUS_REVIEW}
```

**FRONTEND GATE**: Double-check that every frontend ticket being moved
to Review has a QA result (PASS or BLOCKED). If not, go back to Step 4c.

Log cycle summary:
```
[Cycle {N}] COMPLETE — Tickets to Review: #{ISSUE1}, #{ISSUE2} | Flagged: #{ISSUE3} | Retries: {count}
```

Update tracking:
- Add moved tickets to `tickets_to_review`
- Add flagged tickets to `tickets_flagged`
- If zero tickets reached Review this cycle: increment `stale_cycles`
- Otherwise: reset `stale_cycles = 0`
- Append to `cycle_log`

### Stale progress check
If `stale_cycles >= 2`, go to Step 7 (final report) with reason "stuck".

---

## Step 6 — Call PM to refill Ready

Send a message to PM:
```
SendMessage to "pm": "refill — cycle complete, tickets moved to Review"
```

Wait for PM's reply. Parse the signal:
- `READY_TICKETS_AVAILABLE` → increment `cycle_count`, go back to Step 3
- `WAITING` → go to Step 7 (final report)
- `PROJECT_COMPLETE` → go to Step 7 (final report)
- `ALL_BLOCKED` → go to Step 7 (final report)

---

## Step 7 — Shutdown and final report

1. Shut down PM teammate:
   ```
   SendMessage to "pm": {"type": "shutdown_request", "reason": "Run complete"}
   ```
2. Wait for shutdown confirmation, then delete the team `"dev-team-hybrid"`.
3. Output summary:

```
DEV TEAM RUN COMPLETE (hybrid mode)
Outcome: {completed | waiting | stuck | all_blocked | preflight-failed}
Cycles completed: {cycle_count}
Tickets moved to Review: {tickets_to_review or "none"}
Tickets flagged: {tickets_flagged or "none"}

Cycle Log:
| Cycle | Tickets | Sub-agents spawned | Results |
|-------|---------|-------------------|---------|
| 1     | #28     | frontend-dev, test-agent, qa-agent | Review |
| ...   | ...     | ... | ... |
```

---

## Retry limits

- **Max 3 attempts** per ticket for test failures
- **Max 3 attempts** per ticket for QA failures
- Track retry count per issue number throughout all cycles

After max retries for a ticket:
1. Move ticket back to Ready:
   ```bash
   gh project item-edit --project-id ${PROJECT_ID} \
     --id {ITEM_ID} \
     --field-id ${STATUS_FIELD_ID} \
     --single-select-option-id ${STATUS_READY}
   ```
2. Post comment on the issue:
   ```bash
   gh issue comment {ISSUE} --body "Blocked after 3 failed attempts — needs human review.
   Last failure: {error_summary}"
   ```
3. Add to `tickets_flagged`, skip this ticket, continue with others

---

## Rules

- The main thread acts as dev-lead — it NEVER implements code, edits
  source files, runs tests, or creates PRs directly
- ALL implementation work goes to sub-agents via the Agent tool
- All sub-agents MUST use `subagent_type: "general-purpose"`
- Always read agent prompt files (`.claude/agents/dev/*.md`) before
  spawning sub-agents
- PM is the only teammate — it handles board analysis and Ready slot management
- NEVER skip QA for frontend PRs
