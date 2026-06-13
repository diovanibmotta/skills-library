# ship-task

Closes out an implementation: runs tests, performs a local code review, commits remaining work, promotes the PR from draft to ready for review, and moves project board cards.

## Trigger

```
/ship-task
```

## What it does

1. Loads task metadata from the cache written by `/start-task`
2. Auto-detects the project stack (Java/Maven, Angular, Node, Go, Rust, Python, etc.)
3. Runs the appropriate test suite and fails fast on regressions
4. Runs the build or type-check command
5. Performs a local code review: correctness, security, code quality, scope hygiene, and stack-specific checks
6. Fixes blockers inline, presents warnings for user decision
7. Commits any remaining changes with a Gitmoji + Conventional Commits message
8. Pushes to origin
9. Updates the PR description with a structured summary derived from commits
10. Promotes the PR from draft to ready for review (with user confirmation)
11. Moves issue and PR cards on the project board to the target column

## Prerequisites

- Branch created by `/start-task`
- Draft PR already exists
- `gh` authenticated

## Usage

Run on the feature/bugfix branch after implementation is complete:

```
/ship-task
```

## Supported stacks

Angular · Node/JS (Jest, Vitest) · Java (Maven, Gradle) · Go · Rust · Python · any Makefile project
