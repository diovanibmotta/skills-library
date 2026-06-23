# create-feature

Creates a cross-repository feature: one parent issue representing the feature, plus linked sub-tasks in each affected repository derived from an implementation plan.

## Trigger

```
/create-feature [implementation plan]
```

## What it does

1. Parses the implementation plan to identify affected repositories and per-repo scope
2. Derives the feature title automatically from the plan
3. Asks for home repository, priority, project, and sub-task repositories
4. Creates the parent feature issue with full implementation plan and affected repo checklist
5. Creates a sub-task issue in each affected repository with the repo-specific scope
6. Links each sub-task to the parent feature via GitHub's sub-issues API
7. Adds the parent feature to the project board and sets Type/Priority fields
8. Uploads any images from the conversation as issue attachments

## Prerequisites

- `gh` authenticated (`gh auth login`)
- `git` repository with a GitHub remote
- `project` scope on the GitHub token (see below)

## GitHub token scopes

| Scope | Required for |
|-------|-------------|
| `repo` | Creating issues across repositories |
| `read:org` | Listing organization repositories |
| `read:project` | Listing available projects |
| `project` | Listing **and** adding items to the project board (recommended) |

To add the `project` scope to your token:

```bash
gh auth refresh -s project
```

> Without `project` (or `read:project`), the project list will be empty and you will see "no projects found". Issues are still created — board assignment is skipped.

## Usage

Pass the implementation plan as arguments:

```
/create-feature add real-time notifications: backend needs SSE endpoint, frontend subscribes via EventSource, BFF forwards events
```

Or with no arguments — the skill asks for the plan interactively.

## Notes

- Sub-issues API requires GitHub Team plan or higher; if unavailable a manual linking warning is shown
- All sub-tasks receive the same title as the parent feature; per-repo scope is in the issue body
- For features spanning more than 4 repos, the skill processes all repos even if only 4 are shown in the question

## Next step

After creating the feature, run `/start-feature <feature-issue-number>` to create branches and draft PRs across all affected repositories.
