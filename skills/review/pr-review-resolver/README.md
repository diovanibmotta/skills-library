# pr-review-resolver

Resolves GitHub PR code review comments systematically: analyzes each comment, generates solution hypotheses, implements fixes, commits per comment, replies to threads with the commit hash, and resolves them individually.

## Trigger

```
/pr-review-resolver <PR-number or GitHub PR URL>
```

## What it does

1. Fetches all open review threads (skips already-resolved ones)
2. Reads the project's architectural context — `CLAUDE.md`, `README.md`, docs
3. Catalogs and triages comments: Change Request, Suggestion, Question, Nitpick, Blocker
4. Presents the open comments and asks which to address
5. For each selected comment:
   - Reads the referenced file and surrounding context
   - Generates ≥3 solution hypotheses with trade-offs
   - Implements the best fix (or escalates genuine ambiguity to the user)
   - Creates a dedicated commit per comment with a Gitmoji + Conventional Commits message
   - Replies to the review thread with the commit hash, tagging the reviewer
   - Resolves the thread via GraphQL mutation immediately after committing
6. Pushes the branch
7. Posts a rollup summary comment on the PR mentioning all reviewers

## Prerequisites

- `gh` authenticated with GraphQL access
- Must be run on the PR branch

## Usage

```
/pr-review-resolver 123
/pr-review-resolver https://github.com/org/repo/pull/123
```

## Notes

- One commit per review comment — full traceability
- Threads are resolved individually as fixes land, not in batch at the end
- Works with CodeRabbit and other bot reviewers
