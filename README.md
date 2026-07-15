# skills-library

Custom skills library — reusable slash commands for AI-assisted development workflows using Claude Code.

## What are Skills?

Skills are custom slash commands (`.md` files) that extend Claude Code with reusable, project-agnostic behaviors. Each skill encapsulates a specific workflow, pattern, or domain expertise that can be invoked with `/skill-name` inside any Claude Code session.

## Structure

```
skills/
├── dev/          # Development workflow skills
├── review/       # Code review and quality skills
├── infra/        # Infrastructure and DevOps skills
└── docs/         # Documentation and writing skills
```

## Usage

Copy any skill file to `~/.claude/commands/` (global) or `.claude/commands/` (project-local) and invoke it with `/skill-name` in Claude Code.

## Skills Index

### dev

| Skill | Description |
|-------|-------------|
| [create-task](skills/dev/create-task/) | Create a GitHub issue and add it to the project board |
| [create-subtask](skills/dev/create-subtask/) | Create a GitHub sub-issue linked to a parent issue |
| [start-task](skills/dev/start-task/) | Create branch, initial commit, and draft PR for a GitHub issue |
| [ship-task](skills/dev/ship-task/) | Ship a completed task — tests, review, commit, push, PR ready |
| [implement](skills/dev/implement/) | Generate an implementation plan from a GitHub issue |
| [wiki](skills/dev/wiki/) | Create or edit GitHub wiki pages |

### review

| Skill | Description |
|-------|-------------|
| [pr-review-resolver](skills/review/pr-review-resolver/) | Resolve PR review comments with per-comment commits |
| [clean-arch-expert](skills/review/clean-arch-expert/) | Clean Architecture advisor for Java/Spring Boot + Modulith |
| [java-junit-mockito-tests](skills/review/java-junit-mockito-tests/) | Generate Java unit tests with JUnit 5 + Mockito |
| [spec-review](skills/review/spec-review/) | Discover and review AI-generated SPECs (requirements/design/tasks) |
| [code-review-spec](skills/review/code-review-spec/) | Review implemented code against its originating SPEC |
| [security-audit](skills/review/security-audit/) | Structured security review of a diff/branch/PR — OWASP Top 10, secrets, injection, auth/authz, dependency and infra checks |

### infra

| Skill | Description |
|-------|-------------|
| [setup-angular](skills/infra/setup-angular/) | Set up Node.js, Angular CLI, and project dependencies |
| [terraform-iac](skills/infra/terraform-iac/) | Corporate Terraform conventions — file naming, state isolation per layer/component/cloud-provider, module docs, mandatory init/fmt/validate |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
