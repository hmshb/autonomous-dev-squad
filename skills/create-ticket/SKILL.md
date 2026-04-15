---
name: create-ticket
description: Create or update GitHub tickets in this repo with labels and project-board placement. Usage - /create-ticket [description] [--scope frontend|backend|both] to CREATE, or /create-ticket #N <what to change> to UPDATE an existing ticket.
---

Handles **two modes** on the same invocation surface:

1. **Create mode** (default) — creates one or more new issues with the
   right labels and adds them to the project board in `Backlog`. Defaults
   to a single frontend ticket. Use `--scope both` when a feature cleanly
   splits into a frontend ticket **and** a separate backend ticket.
2. **Update mode** — edits an existing issue's title / body / labels
   and/or moves its project-board status. Triggered when the first token
   of the args is an issue reference like `#124` or a GitHub issue URL.

---

## Configuration (REQUIRED — replace before first run)

This skill uses placeholder tokens. See `docs/setup.md` at the repo root
for instructions on discovering your project's actual values using the
`gh` CLI and substituting them in.

## Project constants (hard-coded — do not re-query)

| Field           | Value                  |
|-----------------|------------------------|
| Repo            | `${REPO}`              |
| Owner           | `${OWNER}`             |
| Project number  | `${PROJECT_NUMBER}`    |
| Project ID      | `${PROJECT_ID}`        |
| Status field ID | `${STATUS_FIELD_ID}`   |
| Frontend label  | `frontend`             |
| Backend label   | `backend`              |

### Status option IDs

| Status      | Option ID                 |
|-------------|---------------------------|
| Backlog     | `${STATUS_BACKLOG}`       |
| Ready       | `${STATUS_READY}`         |
| In Progress | `${STATUS_IN_PROGRESS}`   |
| Review      | `${STATUS_REVIEW}`        |
| Done        | `${STATUS_DONE}`          |

If any ID above drifts, rediscover with:
```bash
gh project field-list ${PROJECT_NUMBER} --owner ${OWNER}
gh api graphql -f query='query{node(id:"'${STATUS_FIELD_ID}'"){... on ProjectV2SingleSelectField{options{id name}}}}'
```

---

## Step 0 — Parse the invocation & detect mode

Look at the first non-whitespace token of the args:

- If it matches `#<number>` (e.g. `#124`) **or** looks like a GitHub issue
  URL (`https://github.com/${REPO}/issues/<N>`) → **UPDATE MODE**. The
  rest of the args describe what to change.
- Otherwise → **CREATE MODE**. Parse `--scope` flag and use the whole
  remaining text as the description.

**Synonyms for `--scope both`:** `both`, `fullstack`, `fe+be`, `fe-and-be`.

If the user's intent is ambiguous (e.g. they mention `#124` inside a
create-style description as a dependency reference, not an update
target), ASK before doing anything.

---

# CREATE MODE

## Step 1 — Draft the ticket content

### Title convention
- Frontend → `[FE] <short action-oriented summary>`
- Backend → `[BE] <short action-oriented summary>`
- For `--scope both`, FE and BE titles should clearly reference the same
  feature but describe each half's scope.

Keep titles under ~80 characters. Avoid fluff like "Implement" or "Add
support for". Start with the noun.

### Body template

```markdown
## Overview
<1–3 sentences on what and why>

## Reference (if the user pointed at a doc)
**Implementation plan: [`<path>`](../blob/main/<path>)**

Agents MUST read this document before coding.

## Scope (frontend only | backend only)
1. <concrete file-level step>
2. <concrete file-level step>
...

## Out of scope
- <things that belong in the paired ticket or follow-ups>

## Depends on (if --scope both and one half blocks the other)
- <link to the paired ticket>

## Acceptance criteria
- [ ] <verifiable outcome 1>
- [ ] <verifiable outcome 2>

depends-on: #<issue-number>, #<issue-number>
```

> The trailing `depends-on:` line is what the PM agent parses to resolve
> dependencies. Omit it for tickets with no dependencies.

**Scope-splitting rules for `--scope both`:**
- The **BE ticket should be mergeable on its own** (no FE dependency).
- The **FE ticket `Depends on:` section should link back to the BE
  ticket** so it's obvious BE must merge first if the FE needs new
  endpoints.
- Never duplicate work across the two tickets — each file in the repo
  should be owned by exactly one of the two tickets.

## Step 2 — Create the issue(s)

Use `gh issue create` with a HEREDOC body. Pass exactly one `--label` per
call. For `--scope both`, create the **BE ticket first** (so the FE body
can reference it in its `Depends on` section), then the FE ticket.

```bash
gh issue create \
  --title "<title>" \
  --label "frontend" \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

## Step 3 — Add each issue to the project and move to Backlog

For each created issue:

```bash
gh project item-add ${PROJECT_NUMBER} --owner ${OWNER} --url <issue-url> --format json
```

Parse the `id` field from the JSON (it looks like `PVTI_...`). Then set
the Status field to Backlog:

```bash
gh project item-edit \
  --id <PVTI_...> \
  --field-id ${STATUS_FIELD_ID} \
  --single-select-option-id ${STATUS_BACKLOG} \
  --project-id ${PROJECT_ID}
```

---

# UPDATE MODE

## Step 1 — Identify what to change

From the user's free text after the issue reference, determine which of
the following apply. Multiple can apply at once:

| Intent signal | What to change |
|---|---|
| "change title to …", "rename to …", "title: …" | **Title** |
| "append …", "add … to body", "note that …", a big block of markdown | **Append to body** |
| "replace body with …", "rewrite the body", "new body: …" | **Replace body** |
| "add backend label", "label it frontend", "relabel as …" | **Labels (add/remove)** |
| "move to ready", "set status to in progress", "mark done" | **Project status** |
| "close this", "close as …" | **Close the issue** (state only, do not delete) |
| "reopen" | **Reopen** |
| Free-form content with no clear verb | Ask: append, replace, or something else? |

When the user's text is a pure update payload (e.g. "add a note that we
need to also update the test file"), default to **append to body** — it's
the least destructive.

## Step 2 — Fetch current state (only when needed)

You don't need current state to edit the title, add/remove a label, or
move status. You DO need it to:

- **Append to body** — fetch and append:
  ```bash
  gh issue view <N> --json body --jq .body
  ```
- **Confirm the issue exists before acting** — always fetch at minimum:
  ```bash
  gh issue view <N> --json number,title,state,labels,url
  ```

## Step 3 — Apply the changes

### Title / body / labels — `gh issue edit`

```bash
# Title
gh issue edit <N> --title "new title"

# Body (full replace). Pipe via --body-file to handle multi-line safely:
gh issue edit <N> --body-file - <<'EOF'
<new body>
EOF

# Body (append) — fetch current body, concatenate, then --body-file:
CURRENT=$(gh issue view <N> --json body --jq .body)
gh issue edit <N> --body-file - <<EOF
$CURRENT

---

<appended section>
EOF

# Labels
gh issue edit <N> --add-label backend
gh issue edit <N> --remove-label frontend
```

### Project-board status — `gh project item-edit`

To move an existing issue between board columns you need the
ProjectV2Item ID (different from the issue number). Resolve it with a
GraphQL query:

```bash
gh api graphql -f query='
  query($number: Int!) {
    repository(owner: "'${OWNER}'", name: "'${REPO##*/}'") {
      issue(number: $number) {
        projectItems(first: 10) {
          nodes {
            id
            project { number }
          }
        }
      }
    }
  }' -F number=<N>
```

Pick the `id` from the node whose `project.number` is `${PROJECT_NUMBER}`.
Then move it:

```bash
gh project item-edit \
  --id <PVTI_...> \
  --field-id ${STATUS_FIELD_ID} \
  --single-select-option-id <status-option-id> \
  --project-id ${PROJECT_ID}
```

Use the status option ID table at the top of this file (Backlog / Ready
/ In Progress / Review / Done).

### Close / reopen

```bash
gh issue close <N> --reason completed   # or: not planned
gh issue reopen <N>
```

---

## Step 4 — Summarize to the user

**Create mode** — output a compact table with one row per created
ticket:

```
| # | Ticket | Label | Status |
|---|---|---|---|
| [#N](url) | [FE] ... | frontend | Backlog |
```

**Update mode** — output a short bullet list of the changes applied
with the issue link at the top:

```
Updated [#N](url):
- Title: "<old>" → "<new>"
- Body: appended N lines
- Labels: +backend, -frontend
- Status: Backlog → Ready
```

Call out anything notable: dependency ordering (for `--scope both`),
references to plan docs/mockups that must be committed/pushed so the
ticket links resolve, and any assumptions made while drafting.

---

## Rules

### Shared (both modes)

- **Never create duplicate tickets.** In create mode, run
  `gh issue list --search "<keyword>" --state open` before creating; if a
  match exists, surface it and ask the user whether to reuse or update it
  instead.
- **Never include the `claude` label** unless the user explicitly asks —
  that label triggers Claude Code automation on the issue.
- **Never assign issues** unless the user explicitly asks.
- **Plan docs and mockups must be committed and pushed** for ticket links
  to resolve. If the user references an uncommitted file, warn them at
  the end of the summary.

### Create-mode only

- **Always add to Backlog** on creation. Do not put new tickets in Ready
  or any other column — Backlog is the intake column and the `pm` agent
  promotes to Ready.
- **Never create tickets without user-facing content**: if the user's
  description is just a one-liner, draft the body using context from the
  current conversation — don't invent acceptance criteria from nothing.
- If `gh project item-add` or `item-edit` fails on a fresh issue, do NOT
  delete the issue — leave it created and report the failure so the user
  can recover manually.
- If the scope flag is ambiguous or the description spans both FE and BE
  without a clear split, ASK before creating anything.

### Update-mode only

- **Destructive operations require explicit confirmation.** Before any
  of these, summarize exactly what will change and ask for a yes:
  - Replacing (not appending) the body
  - Removing a label that was the ticket's only label
  - Closing an issue
  - Changing the title to something meaningfully different in intent
  (not just a typo fix)
  Appending to body, adding a label, or moving between project columns
  does NOT require confirmation — those are cheap to undo.
- **Never silently drop content.** When replacing a body, first fetch
  the current body and preserve it in your confirmation prompt so the
  user can compare old vs new.
- **Don't touch other tickets** as a side-effect. If the user's request
  implies cross-ticket changes (e.g. "close #120 and create a follow-up"),
  split into explicit steps and confirm each.
- **Preserve the project-board invariant**: if you move a ticket to
  `Done`, also close the underlying issue unless the user said otherwise.
  Conversely, reopening an issue should move it back to at least `Ready`.
