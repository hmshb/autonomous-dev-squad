# Setup

Before you can run `/dev-team`, `/create-ticket`, or any of the board-aware skills, you need to substitute 8 placeholder tokens with your actual GitHub project values. It takes about 5 minutes with the `gh` CLI.

## Prerequisites

```bash
gh auth login    # if you haven't already
gh auth status   # confirm you're authenticated
```

You need a GitHub repo with a **Projects v2 board** containing a `Status` single-select field with five options: `Backlog`, `Ready`, `In Progress`, `Review`, `Done`. If you don't have one yet, create the project at `https://github.com/users/<you>/projects/new` → choose "New project" → "Board" template, then verify the Status column has those 5 options.

## The placeholders

These tokens appear inside the agents and skills — you replace them once after installing.

| Placeholder             | What it is                                                |
|-------------------------|-----------------------------------------------------------|
| `${OWNER}`              | Your GitHub username or org (e.g., `alice` or `acme-inc`) |
| `${REPO}`               | Full `owner/repo` slug (e.g., `alice/my-saas`)            |
| `${PROJECT_NUMBER}`     | Integer project number (shown in the URL `/projects/N`)   |
| `${PROJECT_ID}`         | GraphQL node ID, starts with `PVT_`                       |
| `${STATUS_FIELD_ID}`    | GraphQL node ID for the Status field, starts with `PVTSSF_` |
| `${STATUS_BACKLOG}`     | Option ID (hex string) for the Backlog column             |
| `${STATUS_READY}`       | Option ID for the Ready column                            |
| `${STATUS_IN_PROGRESS}` | Option ID for the In Progress column                      |
| `${STATUS_REVIEW}`      | Option ID for the Review column                           |
| `${STATUS_DONE}`        | Option ID for the Done column                             |

## Step 1 — Find your project ID and number

```bash
gh project list --owner <your-username>
```

Output looks like:
```
NUMBER  TITLE              STATE  ID
1       my dev board       open   PVT_kwHOAbCdEf1234
2       roadmap            open   PVT_kwHOAbCdEf5678
```

Note the `NUMBER` → `${PROJECT_NUMBER}` and `ID` → `${PROJECT_ID}` of the project you want to use.

## Step 2 — Find the Status field ID

```bash
gh project field-list <project-number> --owner <your-username>
```

Look for the field named `Status` (it's the default on board templates). Note its ID — it starts with `PVTSSF_`. That's your `${STATUS_FIELD_ID}`.

## Step 3 — Find the status option IDs

```bash
gh api graphql -f query='
  query {
    node(id: "<status-field-id-from-step-2>") {
      ... on ProjectV2SingleSelectField {
        options { id name }
      }
    }
  }'
```

Output:
```json
{
  "data": {
    "node": {
      "options": [
        { "id": "a1b2c3d4", "name": "Backlog" },
        { "id": "e5f6a7b8", "name": "Ready" },
        { "id": "c9d0e1f2", "name": "In Progress" },
        { "id": "a3b4c5d6", "name": "Review" },
        { "id": "e7f8a9b0", "name": "Done" }
      ]
    }
  }
}
```

*(the IDs above are fake — your real ones will be unique 8-character hex strings)*

Map each option to its placeholder:
- `Backlog` → `${STATUS_BACKLOG}`
- `Ready` → `${STATUS_READY}`
- `In Progress` → `${STATUS_IN_PROGRESS}`
- `Review` → `${STATUS_REVIEW}`
- `Done` → `${STATUS_DONE}`

## Step 4 — Substitute the tokens

The placeholders appear in these files:
- `.claude/agents/dev/pm-agent.md`
- `.claude/agents/dev/team-lead-agent.md`
- `.claude/skills/create-ticket/SKILL.md`
- `.claude/skills/dev-team-hybrid/SKILL.md`

### macOS / Linux

```bash
cd your-project/.claude

# Set your values here (from Steps 1–3)
OWNER="alice"
REPO="alice/my-saas"
PROJECT_NUMBER="1"
PROJECT_ID="PVT_kwHOAbCdEf1234"
STATUS_FIELD_ID="PVTSSF_lAHOAbCdEf1234"
STATUS_BACKLOG="a1b2c3d4"
STATUS_READY="e5f6a7b8"
STATUS_IN_PROGRESS="c9d0e1f2"
STATUS_REVIEW="a3b4c5d6"
STATUS_DONE="e7f8a9b0"

# Files to edit
FILES=(
  agents/dev/pm-agent.md
  agents/dev/team-lead-agent.md
  skills/create-ticket/SKILL.md
  skills/dev-team-hybrid/SKILL.md
)

# macOS: use `sed -i ''`. Linux: use `sed -i`.
SED_INPLACE=(-i '')   # macOS
# SED_INPLACE=(-i)    # Linux (uncomment this line and comment the one above)

for f in "${FILES[@]}"; do
  sed "${SED_INPLACE[@]}" "s|\\\${OWNER}|$OWNER|g"                      "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${REPO}|$REPO|g"                        "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${PROJECT_NUMBER}|$PROJECT_NUMBER|g"    "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${PROJECT_ID}|$PROJECT_ID|g"            "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${STATUS_FIELD_ID}|$STATUS_FIELD_ID|g"  "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${STATUS_BACKLOG}|$STATUS_BACKLOG|g"    "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${STATUS_READY}|$STATUS_READY|g"        "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${STATUS_IN_PROGRESS}|$STATUS_IN_PROGRESS|g" "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${STATUS_REVIEW}|$STATUS_REVIEW|g"      "$f"
  sed "${SED_INPLACE[@]}" "s|\\\${STATUS_DONE}|$STATUS_DONE|g"          "$f"
done
```

### Windows (PowerShell)

```powershell
cd your-project\.claude

$values = @{
  "OWNER"             = "alice"
  "REPO"              = "alice/my-saas"
  "PROJECT_NUMBER"    = "1"
  "PROJECT_ID"        = "PVT_kwHOAbCdEf1234"
  "STATUS_FIELD_ID"   = "PVTSSF_lAHOAbCdEf1234"
  "STATUS_BACKLOG"    = "a1b2c3d4"
  "STATUS_READY"      = "e5f6a7b8"
  "STATUS_IN_PROGRESS"= "c9d0e1f2"
  "STATUS_REVIEW"     = "a3b4c5d6"
  "STATUS_DONE"       = "e7f8a9b0"
}

$files = @(
  "agents\dev\pm-agent.md",
  "agents\dev\team-lead-agent.md",
  "skills\create-ticket\SKILL.md",
  "skills\dev-team-hybrid\SKILL.md"
)

foreach ($f in $files) {
  $content = Get-Content $f -Raw
  foreach ($k in $values.Keys) {
    $content = $content.Replace("`${$k}", $values[$k])
  }
  Set-Content $f -Value $content -NoNewline
}
```

### Manual (any OS)

Open each file in the list and find-and-replace `${TOKEN}` → your value. There are 10 tokens across 5 files. Takes about 2 minutes.

## Step 5 — Verify no placeholders remain

```bash
grep -rE '\$\{(OWNER|REPO|PROJECT_|STATUS_)' agents/ skills/ && echo "STILL HAS PLACEHOLDERS" || echo "clean"
```

If you see `clean`, you're done. If any files still contain `${...}` tokens, re-run the substitution for the ones you missed.

## Step 6 — Enable Claude Code's teams flag

The dev-team skills require the experimental teams API. Copy `examples/settings.json` to `.claude/settings.json` (or merge its `env` block into your existing one):

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

## Step 7 — Try it

Create a test issue on your board (label it `frontend` or `backend`), drop it in Backlog, then from Claude Code in your project:

```
/dev-team
```

The PM agent should promote your test ticket to Ready within a few seconds, dev-lead should spawn a dev sub-agent, and you'll see cycle logs scroll. If anything looks wrong, the most common problems are:

- **PM says "0 unblocked tickets"** — either your test issue isn't in Backlog, its label isn't `frontend` or `backend`, or it has a `depends-on:` line pointing at a non-Done ticket.
- **Dev-lead says `FATAL: Agent tool missing`** — the teammate-Agent-tool bug fired. Try `/dev-team-hybrid` instead (it makes the main thread the dev-lead, which sidesteps the bug).
- **`gh` command errors** — double-check your `${OWNER}` and `${PROJECT_NUMBER}` values; run `gh project item-list <number> --owner <owner>` manually to confirm they work.

Welcome to the team.
