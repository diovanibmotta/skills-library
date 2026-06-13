# create-task

Creates a GitHub issue and adds it to the project board. Works in any GitHub repository.

## Trigger

```
/create-task [optional title]
```

## What it does

1. Detects the current repository context and available GitHub Projects
2. Derives the issue title from arguments, git commits, or branch name
3. Asks for task type, base branch, priority, and project assignment
4. Detects native GitHub issue types (Feature / Bug / Task) and maps them automatically
5. Creates the issue with the correct label and assignee
6. Uploads any images from the conversation as issue attachments
7. Adds the issue to the selected project board and sets Type/Priority fields

## Prerequisites

- `gh` authenticated (`gh auth login`)
- `git` repository with a GitHub remote

## Usage

```
/create-task add weekly view navigation to calendar
```

Or with no arguments — title is inferred from recent commits or the current branch name.

## Next step

After creating the issue, run `/start-task <issue-number>` to create the branch and draft PR.
