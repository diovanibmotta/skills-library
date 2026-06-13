# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A library of reusable Claude Code skills — custom slash commands (`.md` files) that encapsulate AI-assisted development workflows. No build system, no test runner, no dependencies. Pure markdown + YAML.

## Structure

```
skills/
├── dev/          # Development workflow skills (create-task, start-task, ship-task, implement, wiki, create-subtask)
├── review/       # Code review and quality skills (pr-review-resolver, clean-arch-expert, java-junit-mockito-tests)
├── infra/        # Infrastructure/DevOps skills (setup-angular)
└── docs/         # Documentation skills (currently empty)
```

Each skill lives in its own folder with two files:
- `<skill-name>.md` — the skill prompt (YAML frontmatter + instructions to Claude)
- `install.yaml` — metadata: name, version, description, category, trigger, destinations, requirements
- `README.md` — human-readable description (optional but expected)

## Skill File Format

### `<skill-name>.md`

```markdown
---
description: One-line description used by Claude to decide when to auto-trigger.
---

# Skill Title

Instructions written for Claude to follow when invoked.
```

The frontmatter has only `description`. The body is imperative — instructions to Claude, not docs for humans.

### `install.yaml`

```yaml
name: skill-name
version: 1.0.0
description: Short description
category: dev|review|infra|docs
trigger: /skill-name
author: diovanibmotta
license: MIT

install:
  source: skill-name.md
  destinations:
    global:
      windows: "%USERPROFILE%\.claude\commands\skill-name.md"
      unix: "~/.claude/commands/skill-name.md"
    local: ".claude/commands/skill-name.md"

requirements:
  - name: gh
    description: GitHub CLI — https://cli.github.com
```

## Installing a Skill

Copy the `.md` file to `~/.claude/commands/` (global) or `.claude/commands/` (project-local). The skill is then available as `/skill-name` in any Claude Code session.

## Commit Convention

**Required:** Conventional Commits + Gitmoji. semantic-release runs on CI and blocks non-conforming commits.

Format: `<emoji> <type>(<scope>): <description>`

| Emoji | Type | When |
|-------|------|------|
| ✨ | `feat` | New skill |
| 🐛 | `fix` | Bug fix in existing skill |
| 📝 | `docs` | README, CONTRIBUTING, inline docs |
| ♻️ | `refactor` | Skill rewrite, no behavior change |
| 🔧 | `chore` | Config, CI, tooling |
| 🔥 | `chore` | Remove skill or dead content |
| 💥 | `feat!` | Breaking change in skill interface |

Breaking changes: add `!` after type + `BREAKING CHANGE:` footer.

Example: `✨ feat(dev): add ship-task skill for automated PR shipping`

## Adding a New Skill

1. Create folder under the correct category: `skills/<category>/<skill-name>/`
2. Write `<skill-name>.md` with YAML frontmatter (`description` only) + instructions body
3. Write `install.yaml` with correct metadata and destinations
4. Write `README.md` (optional but recommended)
5. Update Skills Index table in `README.md` at repo root
6. Commit with correct Gitmoji + Conventional Commits format

## PR Checklist

- Skill tested locally in a Claude Code session
- `README.md` Skills Index updated
- Skill placed in correct category folder
- File named `skill-name.md` matching the `name:` field in `install.yaml`
- All commits follow Conventional Commits + Gitmoji format
