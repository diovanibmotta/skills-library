---
description: Ship a completed task — runs tests, performs local code review, commits & pushes, updates the PR from draft to ready, and moves GitHub Project board cards. Works in any language or framework. Run after implementation is complete on a feature/bugfix branch.
---

# Workflow: Ship Task

Closes out an implementation. Validates quality gates, reviews code, commits remaining work, promotes the PR to ready for review, and moves the project board cards.

**Prerequisite:** branch was created by `/create-task`. A draft PR and linked issue must already exist.

---

## Step 1 — Detect context

### 1.1 — Get current branch

```bash
git branch --show-current
```

Store as `currentBranch`.

### 1.2 — Try to load cached task metadata

Before calling GitHub, attempt to read the JSON file written by `/create-task`:

**File path:** `~/.claude/tasks/{{repo}}/{{currentBranch}}.json`
- On Windows: `C:\Users\<user>\.claude\tasks\{{repo}}\{{currentBranch}}.json`
- `repo` is the repository name (e.g. `fluxup-bff`) — get it from `gh repo view --json name --jq '.name'` if not yet known.

Use the **Read tool** (not shell) to read the file. If the file exists and is valid JSON, extract:

| JSON key | Store as |
|---|---|
| `owner` | `owner` |
| `repo` | `repo` |
| `issueNumber` | `issueNumber` |
| `branch` | *(confirms `currentBranch`)* |
| `baseBranch` | `baseBranch` |
| `prUrl` | `prUrl` |
| `prNumber` | `prNumber` |
| `title` | `prTitle` |
| `type` | `taskType` |
| `username` | `ghUsername` |
| `project` | `projectName` |
| `projectNumber` | `projectNumber` |

**Cache hit** — all fields above are now available without GitHub API calls. Skip to step 1.3.

**Cache miss** (file does not exist or cannot be parsed) — run the full detection:

```bash
gh repo view --json owner,name,defaultBranchRef
gh pr view --json number,title,url,isDraft,baseRefName,headRefName,closingIssuesReferences,body 2>/dev/null
gh project list --owner "@me" --format json 2>/dev/null
gh project list --owner "{{owner}}" --format json 2>/dev/null
```

Deduplicate projects by number. Extract each project's `number`, `title`, `id` (node ID).

Store: `owner`, `repo`, `defaultBranch`, `prNumber`, `prUrl`, `baseBranch`, `issueNumber` (from `closingIssuesReferences[0].number`), `projects`.

If `issueNumber` cannot be inferred from `closingIssuesReferences`: try to extract it from the PR body (`Closes #N`) or the branch name pattern.

### 1.3 — Check current PR draft status

Regardless of cache hit or miss, always fetch current draft status (it may have changed since `/create-task` ran):

```bash
gh pr view {{prNumber}} --repo {{owner}}/{{repo}} --json isDraft --jq '.isDraft'
```

Store as `prIsDraft`.

**If no PR exists for the current branch:** stop and tell the user to run `/create-task` first to create a branch, issue, and draft PR.

---

## Step 2 — Detect project type and commands

Inspect the repository root to identify the stack and resolve the correct test/build commands.

```bash
ls -1
cat package.json 2>/dev/null
cat pom.xml 2>/dev/null | head -20
cat build.gradle 2>/dev/null | head -20
cat Cargo.toml 2>/dev/null | head -10
cat go.mod 2>/dev/null | head -5
cat pyproject.toml 2>/dev/null | head -20
cat requirements.txt 2>/dev/null | head -5
cat Makefile 2>/dev/null | head -30
```

Also read all project documentation files at the repo root and common doc directories:

```bash
cat CLAUDE.md 2>/dev/null
cat README.md 2>/dev/null
find . -maxdepth 3 -name "*.md" \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  -not -path "*/dist/*" \
  -not -path "*/target/*" \
  | head -30
```

Read every `.md` file found — especially under `docs/`, `architecture/`, `.agents/`, or `adr/` directories. These files carry architectural decisions, conventions, commit rules, test commands, and patterns that override the generic defaults below.

**Detection rules (first match wins):**

| Detected file / marker | Stack | Test command | Build / type-check command |
|---|---|---|---|
| `angular.json` | Angular | `npx ng test --watch=false --browsers=ChromeHeadless` | `npm run build` |
| `package.json` with `jest` or `vitest` dep | Node/JS | `npm test -- --watchAll=false` | `npm run build 2>/dev/null \|\| npm run typecheck 2>/dev/null` |
| `package.json` (generic) | Node/JS | `npm test` | `npm run build 2>/dev/null` |
| `pom.xml` | Java/Maven | `mvn test -q` | `mvn compile -q` |
| `build.gradle` | Java/Gradle | `./gradlew test` | `./gradlew compileJava` |
| `Cargo.toml` | Rust | `cargo test` | `cargo build` |
| `go.mod` | Go | `go test ./...` | `go build ./...` |
| `pyproject.toml` with pytest | Python | `pytest -q` | `python -m py_compile $(git diff {{baseBranch}}...HEAD --name-only \| grep '\.py$')` |
| `requirements.txt` (no pyproject) | Python | `pytest -q 2>/dev/null \|\| python -m unittest discover -q` | *(skip — interpreted)* |
| `Makefile` with `test` target | Generic | `make test` | `make build 2>/dev/null` |

If no stack is detected or no test command applies: skip Step 3, note it in the final summary.

**Command override priority:** explicit commands in `CLAUDE.md` or `README.md` always override the detection table above.

---

## Step 3 — Run tests

Run the detected test command. Capture full output.

Show a summary: total / passed / failed / skipped.

### Handle failures

If any test fails:

1. Read the failing test file(s) identified in the output.
2. Read the source file(s) they test.
3. Identify whether the failure is a regression from this branch's changes or a pre-existing failure.
4. Fix the code (or the test expectation if it is demonstrably wrong) and re-run.
5. Repeat until all tests pass.

If a failure cannot be resolved with reasonable effort: explain what is failing, classify it (regression vs pre-existing), and ask the user how to proceed before continuing.

**Gate: do not proceed to Step 4 until all tests pass (or the user explicitly chooses to continue with known failures).**

---

## Step 4 — Build / type-check

Run the detected build or type-check command.

Gate: must succeed with no new errors. Pre-existing warnings (bundle size budgets, deprecation notices) are acceptable — do not count them as failures.

---

## Step 5 — Local code review

Identify all files changed on this branch vs the base:

```bash
git diff {{baseBranch}}...HEAD --name-only
```

Read every changed file. Perform a thorough review. Apply checks appropriate to the detected stack, plus the universal checks below.

### Universal checks (all stacks)

**Correctness**
- Logic errors, off-by-one, null/undefined/nil handling
- Edge cases: empty collections, null inputs, missing optional fields
- Async / concurrency: unhandled errors, race conditions, missing awaits

**Security**
- No hardcoded credentials, tokens, secrets, or API keys
- No injection vulnerabilities (SQL, shell, HTML, template injection)
- Inputs from external sources are validated at system boundaries

**Code quality**
- No dead code or unused imports
- No unnecessary complexity introduced beyond what the task requires
- Types are explicit — avoid `any` / `interface{}` / `object` where a concrete type fits
- Comments only where WHY is non-obvious (no WHAT comments, no task references)

**Scope hygiene**
- Changes match the task scope — no unrelated edits snuck in

### Stack-specific checks

Apply project conventions found in `CLAUDE.md`, `README.md`, and any `.md` files read in Step 2 (architecture docs, ADRs, contribution guides). Conventions from these files take precedence over the generic checks below.

For **Angular**: standalone components only; Signal inputs (`input<T>()` / `input.required<T>()`); every user-visible string uses `translate` pipe; all three locale JSON files updated with same keys; `TranslateModule` in `imports[]` of every component using translate pipe.

For **React**: hooks rules (no conditional hook calls); memoization only where profiled as needed; no prop drilling past 2 levels without context or state.

For **Java (Spring)**: no business logic in controllers; repository access only through service layer; no `@Autowired` on fields (prefer constructor injection).

For **Go**: errors handled explicitly (no `_` discards on error returns); no goroutine leaks; exported types and functions have doc comments.

For **Python**: type hints on public functions; no bare `except:`; no mutable default arguments.

For **Rust**: no `unwrap()` or `expect()` in library code; `clippy` warnings addressed.

### Findings

Categorize each finding as:
- **Blocker** — must fix before shipping
- **Warning** — should fix; present to user and wait for decision
- **Nitpick** — note it, do not block

Fix all blockers inline. Present warnings with options. Collect nitpicks for the PR description.

If blockers were fixed: re-run Steps 3 and 4 to confirm nothing regressed.

---

## Step 6 — Stage and commit remaining changes

### 6.1 Check for uncommitted changes

```bash
git status --short
```

If no uncommitted changes exist: skip to Step 7.

### 6.2 Infer commit metadata

- **type**: from branch name prefix (`feature/` → `feat`, `bugfix/` → `fix`, `task/` → infer from PR title)
- **scope**: from the primary changed feature area (e.g. `auth`, `payments`, `calendar`, `api`)
- **issue number**: from Step 1

### 6.3 Commit

Read `CLAUDE.md`, `README.md`, and any contributing/commit convention docs found in Step 2 for the project's commit format. If none is defined, use this default:

```
{{icon}} {{type}}({{scope}}): {{short description, imperative mood, ≤72 chars, no period}}

{{optional body: explain WHY if non-obvious — omit if obvious from subject}}

AFFECT TASK: #{{issueNumber}}
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

**Default icon map:**

| Type | Icon |
|------|------|
| feat | ✨ |
| fix | 🐛 |
| refactor | 🔄 |
| chore | 🔧 |
| docs | 📝 |
| test | 🧪 |
| perf | 🚀 |
| style | 💄 |
| security | 🔒 |

Stage and commit:

```bash
git add -A
git commit -m "{{full commit message}}"
```

---

## Step 7 — Push

```bash
git push origin {{currentBranch}}
```

---

## Step 8 — Update PR description

Fetch all commits on this branch vs base:

```bash
git log {{baseBranch}}..HEAD --oneline
```

Build an updated PR body from the commits, test result, and any nitpicks from the code review:

```markdown
## Summary

{{2-4 bullet points derived from commits on this branch}}

## Motivation

{{Why this change was needed — reference issue #{{issueNumber}}}}

## Test plan

- [x] Tests pass ({{N}} tests)
- [ ] Manual verification on development environment

{{#if nitpicks}}
## Notes

{{list of nitpicks — minor observations not blocking but worth tracking}}
{{/if}}

## Related

Closes #{{issueNumber}}

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

```bash
gh pr edit {{prNumber}} --repo {{owner}}/{{repo}} --body "{{updated-body}}"
```

---

## Step 9 — Mark PR as ready for review

If the PR is currently a draft, ask the user:

> "The PR is still a draft. Mark it as ready for review now?"

If yes:

```bash
gh pr ready {{prNumber}} --repo {{owner}}/{{repo}}
```

If no: skip and note in the final summary that the PR remains a draft.

---

## Step 10 — Move project board cards

### 10.1 Fetch project fields and items

If `projectNumber` is available (from the cached JSON or from Step 1 detection):

```bash
# Fields — need the Status field node ID and option IDs
gh project field-list {{projectNumber}} --owner {{owner}} --format json

# Items — need node IDs for the issue and PR items
gh project item-list {{projectNumber}} --owner {{owner}} --format json --limit 200
```

From `field-list`: find the `Status` field (or the first single-select field). Store its `id` (node ID) and all `options` (`{ id, name }` pairs).

From `item-list`: find:
- Item whose content URL matches `issues/{{issueNumber}}` → store its `id`
- Item whose content URL matches `pull/{{prNumber}}` → store its `id`

If the project node ID is not already known:

```bash
gh project view {{projectNumber}} --owner {{owner}} --format json --jq '.id'
```

### 10.2 Ask where to move the cards

Use `AskUserQuestion` with 2 questions simultaneously:

1. **Issue card** — "Move issue #{{issueNumber}} to which column?"
   - One option per Status column (use real names from `field-list`)
   - Last option: "Keep current / Skip"

2. **PR card** — "Move the PR to which column?"
   - Same Status column options
   - Last option: "Keep current / Skip"

### 10.3 Move the cards

For each card where the user did not select "Keep current / Skip":

```bash
gh project item-edit \
  --project-id {{projectNodeId}} \
  --id {{itemNodeId}} \
  --field-id {{statusFieldNodeId}} \
  --single-select-option-id {{targetOptionId}}
```

Run both updates in parallel if both are being moved.

---

## Step 11 — Final summary

```
✅ Task shipped
─────────────────────────────────────────
Stack     {{detected stack}}
Tests     {{N}} passed, 0 failed
Build     ✅ clean
Review    ✅ {{N}} blockers fixed, {{N}} warnings addressed
Commit    {{short hash}} — {{commit subject}}
PR        {{prUrl}} [{{READY FOR REVIEW or DRAFT}}]
Issue     #{{issueNumber}} → {{target column or "unchanged"}}
PR card   → {{target column or "unchanged"}}
─────────────────────────────────────────
Next: wait for code review. Use /pr-review-resolver to address review comments.
```

---

## Notes

- **Gate order is strict:** tests → build → review → commit → push → PR → board. Do not reorder.
- If no test command is detected, skip Steps 3–4 and note it in the summary.
- If no project is linked to the issue or PR, skip Step 10 silently.
- If the PR was already marked ready for review before this command ran, skip `gh pr ready`.
- If there are no uncommitted changes, skip the commit in Step 6 but still push in Step 7 to ensure remote is up to date.
- Never force-push or amend published commits.
- The `AFFECT TASK` footer in commits is required by default — override only if `CLAUDE.md` specifies a different convention.
- If tests are failing but the user confirms they are pre-existing (not regressions), note them in the PR description under **Notes** and proceed.
