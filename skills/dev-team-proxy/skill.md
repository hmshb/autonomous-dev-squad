---
name: dev-team-proxy
description: Dev team with PM + dev-lead as teammates. Main thread acts as spawn proxy for dev-lead's Agent tool calls.
---

Run the dev team in proxy mode: both PM and dev-lead run as autonomous
teammates. The main thread acts as a thin spawn proxy — it only spawns
sub-agents when dev-lead requests them and relays results back. Dev-lead
keeps all orchestration logic (cycles, retries, board moves, decisions).

This avoids the known bug where teammates don't receive Agent tool access,
while keeping the full two-teammate team structure.

---

## Step 0 — Preflight checks (MANDATORY)

Verify all required agent prompt files exist:
```bash
for f in \
  .claude/agents/dev/pm-agent.md \
  .claude/agents/dev/team-lead-proxy.md \
  .claude/agents/dev/frontend.md \
  .claude/agents/dev/backend.md \
  .claude/agents/dev/test-agent.md \
  .claude/agents/dev/qa-agent.md; do
  [ -f "$f" ] && echo "OK: $f" || echo "MISSING: $f"
done
```

If ANY file is MISSING, stop immediately:
```
PREFLIGHT FAILED: The following required files are missing:
  - {list missing files}
Aborting. Fix the missing files and re-run the skill.
```

---

## Step 1 — Create team and spawn teammates

1. Create a team named `"dev-team-proxy"`.
2. Run `sleep 3`.
3. Read the full contents of:
   - `.claude/agents/dev/pm-agent.md`
   - `.claude/agents/dev/team-lead-proxy.md`
4. Spawn both teammates IN PARALLEL, both as `subagent_type: "general-purpose"`,
   `mode: "bypassPermissions"`, `run_in_background: true`:

   **PM** (name: `"pm"`, team_name: `"dev-team-proxy"`):
   - Use the FULL content of `pm-agent.md` as the prompt
   - But replace the team name references: PM should send messages to
     `"dev-lead"` (the dev-lead teammate, not the main thread)
   - Append: `BEGIN NOW: Run startup workflow.`

   **Dev-Lead** (name: `"dev-lead"`, team_name: `"dev-team-proxy"`):
   - Use the FULL content of `team-lead-proxy.md` as the prompt
   - Append: `BEGIN NOW: Execute Step 0 — run the echo command, verify
     tool access, then send your ENV CHECK result to "team-lead" via
     SendMessage.`

---

## Step 2 — Verify startup

Wait for dev-lead's ENV CHECK message (up to 5 minutes).

**If `ENV CHECK OK`** → proceed to Step 3.

**If FATAL or no response** → enter retry sequence (max 3 attempts):
1. Shut down both teammates
2. `sleep 5`
3. Delete and recreate team
4. `sleep 3`
5. Re-spawn both teammates
6. Wait for ENV CHECK again

After 3 failed attempts:
```
SPAWN FAILED after 3 attempts.
Recommended: wait 2-3 minutes and re-run /dev-team-proxy.
```
Shut down all teammates, delete team, stop.

---

## Step 3 — Spawn proxy loop (MAIN THREAD LOGIC)

You are now the spawn proxy. Your ONLY job is to:
1. Receive spawn/rebase requests from dev-lead
2. Execute them
3. Send results back to dev-lead

You do NOT make decisions about what to spawn, when to retry, or how to
manage the board. Dev-lead makes all those decisions.

### Handling SPAWN_REQUEST messages

When dev-lead sends a message containing `SPAWN_REQUEST`, parse it:

```
SPAWN_REQUEST
type: {frontend-dev | backend-dev | test-agent | qa-agent}
issue: #{ISSUE_NUMBER}
pr: #{PR_NUMBER}
mode: {new | fix}
fix_details: {details}
---
{additional context}
```

Multiple requests in one message are separated by `===`.

For each spawn request:

1. **Read the appropriate agent prompt file** (if not already cached this cycle):
   - `frontend-dev` → `.claude/agents/dev/frontend.md`
   - `backend-dev` → `.claude/agents/dev/backend.md`
   - `test-agent` → `.claude/agents/dev/test-agent.md`
   - `qa-agent` → `.claude/agents/dev/qa-agent.md`

2. **Spawn the sub-agent** using the `Agent` tool:
   - `subagent_type: "general-purpose"`
   - `mode: "bypassPermissions"`
   - For dev agents: `isolation: "worktree"`
   - prompt: FULL content of the agent prompt file + issue/PR numbers +
     any additional context from the request

   If the message contains multiple `SPAWN_REQUEST` blocks separated by
   `===`, spawn ALL of them **in parallel** (multiple Agent calls in one
   message).

3. **Send results back to dev-lead** in this format:
   ```
   SPAWN_RESULT
   type: {agent type}
   issue: #{ISSUE_NUMBER}
   status: {DONE | BLOCKED | FIXED | FAIL}
   pr: #{PR_NUMBER}
   details: {the sub-agent's full return message}
   ```

   If multiple agents were spawned in parallel, combine all results in
   one message separated by `===`.

4. **Shut down the sub-agent immediately** after relaying its result.
   Send a `shutdown_request` to each sub-agent as soon as you have
   sent the SPAWN_RESULT to dev-lead. Do NOT leave completed sub-agents
   running — they accumulate and cause cleanup problems.

### Handling REBASE_REQUEST messages

When dev-lead sends a message containing `REBASE_REQUEST`:

```
REBASE_REQUEST
pr: #{PR_NUMBER}
```

Execute:
```bash
gh pr checkout {PR_NUMBER}
git fetch origin main
git rebase origin/main
git push --force-with-lease
```

Send result back to dev-lead:
```
REBASE_RESULT
pr: #{PR_NUMBER}
status: {OK | CONFLICT}
details: {output or error}
```

### Handling other messages from dev-lead

If dev-lead sends a message that is NOT a `SPAWN_REQUEST` or
`REBASE_REQUEST`, it's informational. No action needed.

### Handling PM messages

PM communicates with dev-lead directly — you should NOT see PM messages.
If PM messages you by mistake, ignore it.

---

## Step 4 — Monitor (while proxying)

While running the proxy loop, also monitor for issues:

- If dev-lead goes silent for more than 10 minutes → send:
  ```
  Watchdog ping: You have been idle for 10 minutes. Report your current status.
  ```
- Two consecutive unanswered watchdog pings → abort:
  ```
  WATCHDOG TIMEOUT: dev-lead is unresponsive. Shutting down team.
  ```
  Go to Step 5.

---

## Step 5 — Shutdown

When dev-lead produces its Final Report (sent as a message to you)
OR the watchdog triggers:

1. Shut down both teammates (`"pm"` and `"dev-lead"`)
2. Wait for shutdown confirmations
3. Delete the team `"dev-team-proxy"`
4. Output summary:
   ```
   DEV TEAM RUN COMPLETE (proxy mode)
   Outcome: {completed | aborted | watchdog-timeout | preflight-failed}
   Cycles completed: {N from dev-lead's report}
   Tickets moved to Review: {list from dev-lead's report, or "none"}
   Tickets flagged: {list from dev-lead's report, or "none"}
   Spawn attempts needed: {N}
   ```

---

## Rules for the main thread (spawn proxy)

- You are a THIN RELAY. Do NOT make orchestration decisions.
- Do NOT decide what agents to spawn — only spawn what dev-lead requests.
- Do NOT move tickets on the board — that's dev-lead's job.
- Do NOT message PM — dev-lead handles that relationship.
- Do NOT interfere with the PM ↔ dev-lead communication.
- Always spawn sub-agents with `subagent_type: "general-purpose"`.
- For dev agents, always use `isolation: "worktree"`.
- Always read the agent prompt file before spawning.
- Spawn multiple parallel requests in a SINGLE message with multiple
  Agent tool calls.
- Send results back to dev-lead promptly — do not batch or delay.
- **ALWAYS shut down sub-agents immediately after relaying their result.**
  Send a shutdown_request right after sending the SPAWN_RESULT to dev-lead.
  Never let completed sub-agents idle — they waste resources and create
  cleanup nightmares at end-of-run.
