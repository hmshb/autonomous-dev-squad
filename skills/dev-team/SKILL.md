---
name: dev-team
description: Spin up dev team (PM + team lead) to work on Ready/Backlog tickets
---

Spin up the dev team to work on project board tickets. Follow these steps:

## Step 0 — Preflight checks (MANDATORY — do this before spawning anything)

Verify all required agent prompt files exist:
```bash
for f in \
  .claude/agents/dev/pm-agent.md \
  .claude/agents/dev/team-lead-agent.md \
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

## Step 1 — Create team

Create a team named `"dev-team"`.

Then immediately run:
```bash
sleep 3
```
This gives the teammate snapshot time to initialize before spawning,
avoiding the race condition in teammate initialization.

---

## Step 2 — Spawn teammates

Spawn two teammates, BOTH as `subagent_type: "general-purpose"`:

- **PM** (name: `"pm"`) — read the full contents of `.claude/agents/dev/pm-agent.md` and use it as the prompt
- **Team Lead** (name: `"dev-lead"`) — read the full contents of `.claude/agents/dev/team-lead-agent.md` and use it as the prompt

> **IMPORTANT**: Never use any `subagent_type` other than `"general-purpose"` — this applies to
> both teammates AND any sub-agents they spawn (dev, test, QA). Named agent types like
> `pm-board-manager` or `team-lead-coordinator` do not provide the required tool access.

---

## Step 3 — Verify startup (with retry)

Wait for dev-lead's first message containing its ENV CHECK result (up to 5 minutes).

**If dev-lead responds with `ENV CHECK OK`** → team started successfully, proceed to Step 4.

**If ANY of the following occur:**
- dev-lead responds with `FATAL: Agent tool missing`
- dev-lead produces an internal error
- dev-lead gives no response within 5 minutes

→ **Enter the retry sequence below. Maximum 3 attempts total.**

### Retry sequence
1. Track attempt number (starts at 1, max 3)
2. Shut down both teammates (`"pm"` and `"dev-lead"`)
3. Run:
   ```bash
   sleep 5
   ```
4. Delete and recreate the team `"dev-team"`
5. Run:
   ```bash
   sleep 3
   ```
6. Re-spawn both teammates exactly as in Step 2
7. Wait again for dev-lead's ENV CHECK response (up to 5 minutes)
8. If `ENV CHECK OK` → proceed to Step 4
9. If still failing and attempt < 3 → go back to step 1 of retry sequence
10. If still failing after attempt 3 → abort:
    ```
    SPAWN FAILED after 3 attempts.
    This is likely the race condition bug in teammate initialization.
    Recommended: wait 2-3 minutes and re-run /dev-team.
    If it keeps failing, restart your claude session and try again.
    ```
    Shut down all teammates and delete the team. Stop.

---

## Step 4 — Monitor

Monitor the team while it runs. You are observing only — do not interfere
with the PM/team-lead loop unless:

- dev-lead goes silent for more than 10 minutes mid-cycle → send:
  ```
  Watchdog ping: You have been idle for 10 minutes. Report your current status.
  ```
- dev-lead receives two consecutive watchdog pings with no response → abort:
  ```
  WATCHDOG TIMEOUT: dev-lead is unresponsive. Shutting down team.
  ```
  Shut down both teammates and delete the team.

---

## Step 5 — Shutdown

When dev-lead produces its Final Report OR the watchdog aborts the run:

1. Shut down both teammates (`"pm"` and `"dev-lead"`)
2. Delete the team `"dev-team"`
3. Output a summary:
   ```
   DEV TEAM RUN COMPLETE
   Outcome: {completed | aborted | watchdog-timeout | preflight-failed}
   Cycles completed: {N}
   Tickets moved to Review: {list or "none"}
   Tickets flagged: {list or "none"}
   Spawn attempts needed: {N}
   ```
