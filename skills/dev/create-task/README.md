# create-task

Creates a GitHub issue and adds it to the project board. Works in any GitHub repository.

## Trigger

```
/create-task [optional title]
```

## What it does

1. Detects the current repository context and available GitHub Projects
2. Derives the issue title from arguments, git commits, or branch name
3. Asks for task type, base branch, GitHub issue type, and project assignment
4. Creates the issue with the correct label and assignee
5. Uploads any images from the conversation as issue attachments
6. Adds the issue to the selected project board and sets Type/Priority fields

## Prerequisites

- `gh` authenticated (`gh auth login`)
- `git` repository with a GitHub remote
- `project` scope on the GitHub token (see below)

## GitHub token scopes

| Scope | Required for |
|-------|-------------|
| `repo` | Creating issues, labels |
| `read:project` | Listing available projects |
| `project` | Listing **and** adding items to the project board (recommended) |

To add the `project` scope to your token:

```bash
gh auth refresh -s project
```

> Without `project` (or `read:project`), the project list will be empty and you will see "no projects found". The issue is still created — board assignment is skipped.

## Usage

```
/create-task add weekly view navigation to calendar
```

Or with no arguments — title is inferred from recent commits or the current branch name.

## Next step

After creating the issue, run `/start-task <issue-number>` to create the branch and draft PR.
