# start-feature

Given a parent feature issue number, discovers all linked sub-tasks across repositories, creates a consistent `feature/<slug>` branch in each affected repo, adds an initial commit, and opens a draft PR. Works across any GitHub organization.

## Trigger

```
/start-feature <feature-issue-number>
```

## What it does

1. Fetches the parent feature issue and all linked sub-tasks via GitHub's sub-issues API
2. Derives a consistent branch name from the feature title (e.g. `feature/real-time-notifications`)
3. Creates the same branch in every affected repository using `gh issue develop` to establish Development section links
4. Adds an initial commit in each repo so PRs can be opened
5. Opens a draft PR in each repository with a structured description referencing the parent feature
6. Saves feature metadata to `~/.claude/tasks/{homeRepo}/{branch}.json`

## Branch naming

Feature title is converted to kebab-case, prefixed with `feature/`, and truncated to 50 chars:

```
"add real-time notifications" → feature/add-real-time-notifications
```

The **same branch name** is used across all repositories for consistency.

## Prerequisites

- `gh` authenticated (`gh auth login`)
- Feature and sub-tasks created via `/create-feature`
- Sub-issues feature enabled on the GitHub plan/org

## GitHub token scopes

| Scope | Required for |
|-------|-------------|
| `repo` | Creating branches and PRs across repositories |
| `read:org` | Listing organization repositories |

> No `project` scope needed — this skill does not interact with the project board.

## Usage

```
/start-feature 15
```

Or with a cross-repo reference:

```
/start-feature fluxup-platform/fluxup-backend#15
```

## Notes

- Initial commits are created via GitHub API — no local clone needed for the other repos
- If a branch already exists in a repo, creation is skipped and the skill proceeds to PR creation
- If sub-issues API is unavailable, the skill falls back to prompting for the repo list manually

## Next step

Implement the changes in each repo on the `feature/<slug>` branch, then run `/ship-task` per sub-task when ready for review.
