# Contributing

Contributions welcome — new skills, improvements, and fixes.

## Adding a Skill

1. Fork this repo
2. Create your skill file under the appropriate `skills/` subdirectory
3. Follow the skill format below
4. Update the Skills Index in `README.md`
5. Open a pull request

## Skill File Format

Each skill is a Markdown file with a YAML-like header followed by the prompt body:

```markdown
---
name: skill-name
description: One-line description of what the skill does and when to trigger it.
---

# Skill Title

Detailed instructions for Claude to follow when this skill is invoked.

## Steps

1. Step one
2. Step two
```

### Guidelines

- **Name**: kebab-case, short, descriptive
- **Description**: one sentence — used by Claude to decide when to auto-trigger
- **Body**: write as instructions to Claude, not documentation for humans
- **Scope**: one skill = one focused responsibility
- **Language**: English (for portability)

## Categories

| Folder | Use for |
|--------|---------|
| `skills/dev/` | Development workflows, scaffolding, testing |
| `skills/review/` | Code review, quality, security |
| `skills/infra/` | CI/CD, cloud, containers |
| `skills/docs/` | Documentation, changelogs, ADRs |

## Commit Messages

All commits **must** follow [Conventional Commits](https://www.conventionalcommits.org/) — this repo uses semantic-release to automate versioning and changelogs.

### Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Types

| Type | When to use |
|------|-------------|
| `feat` | New skill added |
| `fix` | Bug fix in an existing skill |
| `docs` | README, CONTRIBUTING, or inline doc changes |
| `refactor` | Skill rewrite with no behavior change |
| `chore` | Repo config, CI, tooling — no skill changes |

### Examples

```
feat(dev): add ship-task skill for automated PR shipping
fix(review): correct checklist order in code-review skill
docs: update skills index with new infra skills
```

### Breaking Changes

Add `!` after the type or a `BREAKING CHANGE:` footer for skills that change their interface:

```
feat(dev)!: rename start-task to begin-task

BREAKING CHANGE: slash command renamed from /start-task to /begin-task
```

> PRs with non-conforming commit messages will be blocked.

## Pull Request Checklist

- [ ] Skill tested locally in a Claude Code session
- [ ] `README.md` Skills Index updated
- [ ] Skill placed in correct category folder
- [ ] File named `skill-name.md` matching the `name:` field
- [ ] All commits follow Conventional Commits format

## Issues

Open an issue to propose a new skill, report a bug, or request improvements.
