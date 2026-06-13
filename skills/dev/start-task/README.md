# start-task

Given a GitHub issue number, creates the branch, makes an initial commit (if changes exist), and opens a draft PR. Works in any GitHub repository.

## Trigger

```
/start-task <issue-number>
```

## What it does

1. Resolves the issue details and infers the task type from its labels
2. Asks for base branch and GitHub Project assignment for the PR card
3. Creates a branch using `gh issue develop` — which links the branch to the issue in the Development section automatically
4. Stages and commits any pending changes with a Gitmoji + Conventional Commits message
5. Pushes the branch and opens a draft PR with a structured description
6. Adds the PR to the project board and sets Type/Priority fields
7. Saves task metadata to `~/.claude/tasks/{repo}/{branch}.json` for use by `/ship-task`

## Branch naming

| Type | Pattern | Example |
|------|---------|---------|
| `feat` | `feature/<kebab-title>` | `feature/user-auth` |
| `fix` | `bugfix/<issue-number>` | `bugfix/42` |
| other | `task/<issue-number>` | `task/42` |

## Prerequisites

- `gh` authenticated (`gh auth login`)
- Issue created via `/create-task`

## Usage

```
/start-task 42
```

## Next step

Implement the changes, then run `/ship-task` to validate, commit, and mark the PR ready.
