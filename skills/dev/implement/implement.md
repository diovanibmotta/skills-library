---
description: Generate an implementation plan for the current branch task, save it as a .md file, and present it for user review and approval. Reads task metadata from /start-task JSON and GitHub issue context. Works in any repository and framework.
---

# Workflow: Implement Changes

Given the current branch (or `$ARGUMENTS` as an explicit branch name), generates a structured implementation plan backed by the GitHub issue context and codebase exploration, saves it to a local markdown file, and presents it for user review, approval, or iteration.

**Prerequisite:** Run `/start-task <issue-number>` first so the branch and task metadata exist.

---

## Step 1 — Resolve branch and load metadata

### 1.1 — Determine branch name

If `$ARGUMENTS` is non-empty, use it as `branchName`.
Otherwise run:

```bash
git branch --show-current
```

Store as `branchName`.

### 1.2 — Load repository context

```bash
gh repo view --json owner,name,defaultBranchRef
gh api user --jq '.login'
```

Store `owner`, `repo`, `defaultBranch`, `ghUsername`.

### 1.3 — Load cached task metadata

Attempt to read the JSON file written by `/start-task` using the **Read tool** (not shell).

**Try these paths in order:**

1. `C:\Users\{{ghUsername}}\.claude\tasks\{{repo}}\{{branchName-slash-to-backslash}}.json`
   - Example: `C:\Users\diova\.claude\tasks\fluxup-backend\bugfix\154.json`
2. `C:\Users\{{ghUsername}}\.claude\tasks\{{repo}}\{{branchName-slash-to-dash}}.json`
   - Example: `C:\Users\diova\.claude\tasks\fluxup-backend\bugfix-154.json`

On Unix/Mac, use `~/.claude/tasks/{{repo}}/{{branchName}}.json` (forward slashes, branch path segments preserved).

**Cache hit** — extract from JSON:

| JSON key | Store as |
|----------|----------|
| `issueNumber` | `issueNumber` |
| `title` | `taskTitle` |
| `type` | `taskType` |
| `baseBranch` | `baseBranch` |
| `prUrl` | `prUrl` |
| `prNumber` | `prNumber` |
| `priority` | `priority` |
| `owner` | `owner` (override if different) |
| `repo` | `repo` (override if different) |

**Cache miss** — attempt to derive from branch name and PR:

```bash
# Try to extract issue number from branch name (e.g. bugfix/154 → 154, task/103 → 103)
echo "{{branchName}}" | grep -oE '[0-9]+$'

# Try to read current PR
gh pr view --json number,title,url,isDraft,baseRefName,closingIssuesReferences 2>/dev/null
```

If issue number cannot be determined:
- If `$ARGUMENTS` looks like a number, use it as `issueNumber`
- Otherwise warn the user:
  > "No task metadata found for branch `{{branchName}}`. Run `/start-task <issue-number>` first, or run `/implement <issue-number>` to pass the issue number directly."
  Then stop.

---

## Step 2 — Fetch full issue context

Run in parallel:

```bash
gh issue view {{issueNumber}} \
  --repo {{owner}}/{{repo}} \
  --json number,title,body,labels,comments
```

Extract:
- `issueTitle` — issue title
- `issueBody` — full markdown body (contains root cause, proposed fix, affected files)
- `issueComments` — array of comment bodies (may contain spec clarifications)

---

## Step 3 — Read project conventions

Using the **Read tool**, read (if they exist):
- `CLAUDE.md` in the working directory root
- `docs/development-guidelines.md` (or equivalent referenced in CLAUDE.md)

Extract and internalize:
- Module/layer structure and naming conventions
- Required Lombok annotations, null-handling patterns
- Test patterns (builders, assertion rules, helper constants)
- Any architecture rules relevant to the affected layer (gateway, use case, facade, etc.)

---

## Step 4 — Explore the codebase

Based on the issue body (root cause, proposed fix, affected files listed), **explore relevant code** using Glob, Grep, and Read tools.

For **each file explicitly mentioned** in the issue body:
1. Locate it (Glob with the class name pattern)
2. Read its full content (Read tool)
3. Identify what exactly needs to change and why

Additionally search for:
- **Test files** for changed classes (Grep for `{{ClassName}}Test`)
- **Interfaces** the changed class implements (Grep for `implements {{Interface}}`)
- **Callers** of the changed class or method (Grep for the class/method name)
- **Configuration files** (`.yml`, `.properties`) if the issue mentions config

**Goal:** collect enough information to write precise, file-specific implementation steps with line-level accuracy.

---

## Step 5 — Generate implementation plan

Build the plan document using the template below. Every section must be filled — do NOT leave template placeholders.

**Determine the plan file path:**

| OS | Path |
|----|------|
| Windows | `C:\Users\{{ghUsername}}\.claude\tasks\{{repo}}\{{branchName-slash-to-dash}}-plan.md` |
| Unix/Mac | `~/.claude/tasks/{{repo}}/{{branchName-slash-to-dash}}-plan.md` |

Example: `C:\Users\diova\.claude\tasks\fluxup-backend\bugfix-154-plan.md`

Save the file using the **Write tool**.

---

### Plan template

```markdown
# Implementation Plan: {{issueTitle}}

**Issue:** [#{{issueNumber}}]({{issueUrl}}) | **Branch:** `{{branchName}}` | **PR:** [#{{prNumber}}]({{prUrl}})
**Type:** `{{taskType}}` | **Priority:** {{priority or "—"}} | **Generated:** {{YYYY-MM-DD}}

---

## Problem

{{Concise description of what is broken or missing. Use language from the issue body. 2–4 sentences.}}

## Root Cause

{{Technical root cause analysis. Name the specific class, method, config, annotation, or data involved.
Explain the causal chain: "X happens because Y, which causes Z."}}

## Solution

{{What will be changed and why — high-level description, not implementation details yet.
Explain the chosen approach and why alternatives were ruled out (if relevant).}}

---

## Files to Modify

| File | Layer | Change Summary |
|------|-------|---------------|
| `path/to/File.java` | infrastructure | Short description of the change |

## Files to Create

| File | Layer | Purpose |
|------|-------|---------|
| *(none)* | — | — |

## Tests to Update / Create

| Test File | Status | What to Test |
|-----------|--------|-------------|
| `path/to/FileTest.java` | existing | Describe what test scenario to add or update |

---

## Implementation Steps

For each step: name the file, describe the change precisely, show before/after code if helpful.

### Step 1 — {{Short imperative title}}

**File:** `path/to/File.java`

{{What to change and why. Reference specific method names, line numbers, annotations, or fields.
Be precise enough that the implementer knows exactly what to type.}}

```java
// Before
old code
// After
new code
```

### Step 2 — {{Short imperative title}}

**File:** `path/to/File.java`

{{Description of change.}}

*(repeat for each step)*

---

## Verification

```bash
# Compile to catch any errors early
mvn clean compile

# Run directly affected tests
mvn test -Dtest=AffectedTestClass

# Run full suite
mvn test
```

## Risks / Notes

- {{Breaking changes, side effects, or things to watch for}}
- *(none)* if no risks identified
```

---

## Step 6 — Present plan for review

Output the full plan content inline in your response (render the markdown), then show:

```
Plan saved → {{plan-file-path}}
```

Then use `AskUserQuestion` with one question:

**"Review the implementation plan:"** (single-select)

- `Approve — proceed to implementation`
- `Request changes — I'll provide feedback`
- `Reject — discard and start over`

---

## Step 7 — Handle user response

### "Approve — proceed to implementation"

Reply:

> **Plan approved.** Implement the changes listed above. When done, run `/ship-task` to run tests, commit, and mark the PR ready for review.

Optionally: begin implementing Step 1 immediately if the changes are unambiguous and scoped to a small number of files (ask the user: "Should I start implementing now?").

### "Request changes — I'll provide feedback"

Ask the user (free-form message, NOT `AskUserQuestion`):

> "What should be changed in the plan? Describe corrections, additions, or clarifications."

Wait for the user's reply. Then:
1. Revise the affected sections of the plan in memory
2. Overwrite the plan file with the updated content using the **Write tool**
3. Output the revised plan inline
4. Return to Step 6 (ask for approval again)

Repeat until the user approves or rejects.

### "Reject — discard and start over"

Reply:

> **Plan discarded.** Branch `{{branchName}}` is still active. Run `/implement` again to generate a fresh plan, or run `/ship-task` to close out without a plan.

Delete the plan file:

```bash
rm "{{plan-file-path}}"
```

---

## Notes

- **Works in any repository and language/framework.** Architecture exploration adapts to what is found in CLAUDE.md and the codebase.
- Plan files are local only — never committed to the repository.
- If the issue body already contains a detailed proposed fix, the plan should validate and expand it with file paths and code snippets, not replace it.
- If `gh` is not authenticated, stop at Step 2 and instruct the user to run `gh auth login`.
