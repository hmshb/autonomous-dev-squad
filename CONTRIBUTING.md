# Contributing

Thanks for your interest in improving `autonomous-dev-squad`.

## Ways to contribute

- **Stack adapters.** The test and QA agents currently assume a FastAPI + Next.js layout. PRs adapting them to other stacks (Go, Rails, Django, Rust, Vue, etc.) are very welcome — either as alternative agent files or as commented sections inside the existing ones.
- **New QA module templates.** The example modules in `agents/qa/system-qa.md` are from a single SaaS. Contribute templates for common app types (e-commerce checkout flow, auth + profile, dashboard + admin, etc.).
- **Teammate stability improvements.** If you find a more reliable way to spawn teammates with Agent tool access, PRs to the `dev-team-*` skills are welcome.
- **Documentation.** Typos, clarifications, extra setup steps for Windows/Linux/macOS differences.
- **Bug reports.** Open an issue with: Claude Code version, OS, the command you ran, and what you expected vs. what happened.

## Development setup

This is a repository of Markdown-based prompts — there is no build step. Clone, edit, test locally in a throwaway project, and PR.

```bash
git clone https://github.com/hmshb/autonomous-dev-squad.git
cd autonomous-dev-squad

# Edit agents/skills as needed

# Test in a real project
cp -r agents skills /path/to/your/test-project/.claude/
cd /path/to/your/test-project
# then run the skill in Claude Code
```

## PR conventions

- **One concern per PR.** If you're both fixing a typo and adding a stack adapter, split them.
- **Update the README** if you add or remove an agent/skill.
- **Keep placeholders consistent.** If you touch a file with `${REPO}`, `${PROJECT_ID}`, etc., don't hardcode your own values — the tokens are the public interface.
- **Describe the test.** In your PR body, explain how you verified the change (e.g., "ran `/dev-team` on a 3-ticket test project, all cycles completed"). This is prompt engineering, not code — there are no unit tests to run, so manual verification notes are the main signal.
- **Avoid scope creep.** Don't refactor unrelated files in the same PR.

## Style

- Markdown agents and skills should be imperative, numbered where appropriate, and testable. "Do X, then Y, then if Z report it back" beats "The agent handles X and Y, and sometimes Z."
- Keep placeholder tokens in the form `${NAME}` — no other conventions.
- If you add configuration, update `docs/setup.md` in the same PR.

## Questions

Open a discussion or issue on GitHub. This project is actively maintained.
