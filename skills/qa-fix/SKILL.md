---
name: qa-fix
description: Instruction-driven QA — test and fix via Playwright based on free text or module name. Usage - /qa-fix [module or free text instruction]
---

Run instruction-driven QA: test and fix autonomously based on the user's
input. Accepts either a module name or free text describing what to test.

**Input parsing:**
- If args match a known module name → run that module's predefined scenarios
- If args are free text (e.g., "the persona chips on generate page aren't
  highlighting on click") → run custom QA targeting the described issue
- If no args → ask the user what to test

> **Customize the module list** in `.claude/agents/qa/system-qa.md` and
> this skill's Step 0 for your project. The default list is an example.

---

## Step 0 — Parse input

Read the user's arguments from the skill invocation.

**Module mode** — args match a module name:
- Set `MODE = module`
- Set `MODULE = {matched name}`
- Set `INSTRUCTION = null`

**Instruction mode** — args are free text:
- Set `MODE = instruction`
- Set `MODULE = null` (infer from context during testing)
- Set `INSTRUCTION = {the user's text}`

---

## Step 1 — Prepare

1. Ensure `qa-reports/` directory exists:
   ```bash
   mkdir -p qa-reports
   ```
2. Determine the branch to test on. Check current branch:
   ```bash
   git branch --show-current
   ```
   Use whatever branch is currently checked out (the user likely has
   the code they want tested already running).

---

## Step 2 — Run QA test agent

Spawn a sub-agent (`subagent_type: "general-purpose"`, `mode: "bypassPermissions"`)
with the system-qa prompt adapted for the current mode.

Read `.claude/agents/qa/system-qa.md` for the base prompt.

**Module mode:** Use the base prompt as-is, providing:
- MODULE: `{MODULE}`
- BRANCH: current branch
- CYCLE: 1

**Instruction mode:** Use the base prompt but REPLACE the module test
scenarios section with the user's instruction. Build a custom test plan:

```
CUSTOM TEST INSTRUCTION:
{INSTRUCTION}

Based on this instruction, determine:
1. Which page(s) to navigate to
2. What actions to perform
3. What to verify (expected behavior)
4. Take screenshots of any issues found

Write results to qa-reports/custom-bugs.md using the same bug report
format. Use "custom" as the module name.
```

Wait for the agent to complete.

---

## Step 3 — Check results

Read the bug report:
- Module mode: `qa-reports/{MODULE}-bugs.md`
- Instruction mode: `qa-reports/custom-bugs.md`

Count OPEN bugs (ignore OBSERVATIONS).

- **0 open bugs** → output "QA PASS — no bugs found" and STOP
- **Open bugs exist** → proceed to Step 4

---

## Step 4 — Run fix agent (autonomous)

Read `.claude/agents/qa/fullstack.md` for the base prompt.

Create a fix branch:
```bash
git checkout -b fix/{MODULE_OR_CUSTOM}-qa-fix 2>/dev/null || git checkout fix/{MODULE_OR_CUSTOM}-qa-fix
```

Spawn a fix sub-agent (`subagent_type: "general-purpose"`, `mode: "bypassPermissions"`)
with the fullstack prompt, providing:
- MODULE: `{MODULE}` or `custom`
- BRANCH: `fix/{MODULE_OR_CUSTOM}-qa-fix`
- CYCLE: current cycle number

The fix agent will:
1. Read the bug report
2. Fix all OPEN bugs
3. Run tests to verify no regressions
4. Commit, push, and create/update a PR

Wait for the agent to complete.

---

## Step 5 — Ask user to restart servers

After the fix agent completes, ask the user to restart the servers on the
fix branch before re-testing.

Wait for user confirmation before proceeding.

---

## Step 6 — Re-test (max 3 cycles)

Track cycle count (started at 1 in Step 2).

After user confirms servers are restarted:
1. Increment cycle count
2. Go back to Step 2, but now:
   - BRANCH = the fix branch
   - CYCLE = current cycle number
3. After Step 3:
   - If 0 open bugs → output "QA PASS after {N} cycles" and STOP
   - If open bugs AND cycle < 3 → go to Step 4 again
   - If open bugs AND cycle = 3 → run one final fix (Step 4), then
     one verification test. Mark done regardless of result.

---

## Step 7 — Final summary

Output:
```
QA-FIX COMPLETE
Mode: {module | instruction}
Target: {MODULE or first 60 chars of instruction}
Cycles: {N}
Result: {PASS | PARTIAL — N bugs remain}
PR: {PR_URL or "none (clean pass)"}
Bugs found: {total}
Bugs fixed: {count}
Bugs remaining: {count}
```

---

## Rules

- ALL testing via Playwright MCP browser tools — no curl/httpx/API testing
- ALL fixes go to a PR — never push to main directly
- Fix and re-test autonomously — no approval gate between cycles
- Only pause for server restarts (Step 5)
- Max 3 fix cycles — don't loop forever
- Sub-agents must use `subagent_type: "general-purpose"` for Playwright access
- Do NOT modify test files unless the test itself is wrong
- Do NOT add features or refactor — only fix reported bugs
- OBSERVATIONS are informational — do NOT trigger fix cycles for them
