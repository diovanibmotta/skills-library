# spec-review

Discovers and reviews AI-generated SPECs (requirements/design/tasks and related files) in any repository, lists which ones still need review, synthesizes the proposed implementation, identifies gaps/contradictions/ambiguities, and generates a review file to send back to the original agent.

## Trigger

```
/spec-review [spec name or path]
```

## What it does

1. Discovers spec folders in the current repository — known agent conventions (Kiro, GitHub Spec Kit) plus a generic heuristic for uncatalogued ones
2. Determines which specs were never reviewed, changed since the last review (via file fingerprint), pending a decision, or already approved
3. Lets the operator pick which spec to review (interactively, or directly if named in the request)
4. Reads and synthesizes the full spec, plus additional project context (`README.md`, `CLAUDE.md`/`AGENTS.md`, architecture docs, CI config)
5. Cross-audits the documents for gaps, contradictions, and ambiguities — each finding becomes an objective question with 2–3 suggested paths forward
6. Generates a versioned review file at `.ai/spec-reviews/<agent>/<spec-slug>/review-<timestamp>.md` with front matter tracking source file fingerprints and review status

## Target stack

Any project, any language, any spec-generating agent (Kiro, GitHub Spec Kit, Cursor, Windsurf, Cline, Claude Code, or ad hoc conventions)

## Usage

```
/spec-review
/spec-review checkout-flow
```

## Notes

- Review files are saved inside the project (`.ai/spec-reviews/`) so history is versioned alongside the code
- Re-running after corrections automatically detects changed source files via fingerprint and treats the spec as pending a new review
- Pairs with the `code-review-spec` skill, which audits the *implementation* against the spec and this review's recorded operator decisions
