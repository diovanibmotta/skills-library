# implement

Generates a structured implementation plan for the current branch task, saves it as a local `.md` file, and presents it for review and approval before any code is written.

## Trigger

```
/implement [branch-name or issue-number]
```

## What it does

1. Resolves the current branch and loads task metadata from the `/start-task` cache
2. Fetches the full GitHub issue (title, body, comments)
3. Reads `CLAUDE.md` and project documentation to internalize conventions
4. Explores the codebase — reads every file explicitly mentioned in the issue, plus tests, interfaces, and callers
5. Generates a plan with: Problem, Root Cause, Solution, Files to Modify/Create, Tests, Implementation Steps, Verification commands, and Risks
6. Saves the plan to `~/.claude/tasks/{repo}/{branch}-plan.md`
7. Presents the plan inline and asks for Approve / Request changes / Reject

## Prerequisites

- Branch created by `/start-task`
- GitHub issue with a meaningful description (root cause, proposed fix, affected files)
- `gh` authenticated

## Usage

```
/implement
```

Or pass an issue number directly if metadata cache is missing:

```
/implement 154
```

## Notes

- Plan files are local only — never committed to the repository
- The skill iterates on the plan until the user approves it
- Works in any repository and framework — architecture exploration adapts to what is found in `CLAUDE.md`
