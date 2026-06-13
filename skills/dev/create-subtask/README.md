# create-subtask

Creates a GitHub issue in any repository and links it as a sub-issue of a parent issue.

## Trigger

```
/create-subtask [optional title]
```

## What it does

1. Detects the current repository and available projects
2. Asks for the parent issue (supports `owner/repo#number` or `#number` format)
3. Derives the subtask title automatically from arguments, commits, or branch name
4. Asks for task type, target repository, priority, and project board
5. Creates the issue with label, assignee, and native GitHub issue type
6. Links the new issue as a sub-issue of the parent via GitHub API
7. Adds the issue to the project board and sets Type/Priority fields

## Prerequisites

- `gh` authenticated (`gh auth login`)
- Sub-issues feature enabled on the GitHub plan/org

## Usage

```
/create-subtask implement payment retry logic
```

When prompted, provide the parent issue: `owner/repo#42` or just `#42`.

## Notes

- The subtask can be created in a different repository than the parent
- If sub-issues are not enabled on the plan, a manual linking warning is shown

## Next step

Run `/start-task <issue-number>` to create the branch and draft PR for the subtask.
