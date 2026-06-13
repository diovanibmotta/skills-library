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

| Skill | Description | Category |
|-------|-------------|----------|
| *(skills will be listed here as they are added)* | | |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
