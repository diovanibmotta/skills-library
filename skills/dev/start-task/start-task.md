---
description: Given a GitHub issue number, creates the branch, initial commit (if changes exist), and draft PR. Run after /create-task. Works in any GitHub repository.
---

# Workflow: Start Task

Given a GitHub issue, creates the branch linked to the issue, optionally commits pending changes, and opens a draft PR. Works in **any** GitHub repository.

Run `/create-task` first to create the issue. Then run `/start-task <issue-number>`.

---

## Step 1 — Detect repository context and resolve issue

Run the following in parallel:

```bash
gh repo view --json owner,name,defaultBranchRef
git branch --show-current
git branch -a --format="%(refname:short)" | sed 's|origin/||' | sort -u | head -40
```

Also fetch the org/user name from the repo and list projects:

```bash
gh project list --owner "@me" --format json 2>/dev/null
gh project list --owner "<org>" --format json 2>/dev/null
```

**Resolve the issue number** from `$ARGUMENTS`:
- If `$ARGUMENTS` contains a number (e.g. `42` or `#42`), use it as `issueNumber`.
- If `$ARGUMENTS` is empty, ask the user: *"Which issue number should I start? (e.g. 42)"*

Fetch the issue details:

```bash
gh issue view {{issueNumber}} \
  --repo {{owner}}/{{repo}} \
  --json number,title,labels,body
```

From the response extract:
- `title` — issue title
- `type` — infer from labels using the reverse label mapping below
- `baseBranch` — look for a line `Base branch: <branch>` in the issue body; fall back to `defaultBranch`

**Reverse label mapping:**

| Label         | Type       |
|---------------|------------|
| `enhancement` | `feat`     |
| `bug`         | `fix`      |
| `refactor`    | `refactor` |
| `chore`       | `chore`    |
| `documentation` | `docs`   |
| `test`        | `test`     |
| `performance` | `perf`     |
| `build`       | `build`    |
| `ci`          | `ci`       |
| `style`       | `style`    |

If no matching label is found, default `type` to `chore`.

Store: `owner`, `repo`, `defaultBranch`, `currentBranch`, `issueNumber`, `title`, `type`, `baseBranch`, list of projects (number + title).

---

## Step 2 — Ask for base branch and project

Ask the user the following questions **all at once** using `AskUserQuestion` with **2 questions**.

**IMPORTANT — `AskUserQuestion` enforces a hard limit of 4 options per question.**

1. **Base branch** (single-select, **max 4 options** — list up to 3 detected branches from Step 1; mark `main` or `develop` as recommended if present; pre-select `baseBranch` resolved in Step 1 if it appears in the list)

2. **GitHub Project** (single-select, **optional**, **max 4 options** — list up to 3 projects detected in Step 1 by title; always include "None / Skip" as first option):
   - `None / Skip`
   - `{{project-title-1}}` (number: {{N}})
   - `{{project-title-2}}` (number: {{N}})
   - `{{project-title-3}}` (number: {{N}}) *(include only if ≤3 projects found)*
   - If no projects were found, show only "None / Skip"

---

## Step 3 — Determine branch name

Apply the naming convention:

| Type              | Pattern                               | Example             |
|-------------------|---------------------------------------|---------------------|
| `feat`            | `feature/<title-in-kebab-case>`       | `feature/user-auth` |
| `fix`             | `bugfix/{{issueNumber}}`              | `bugfix/42`         |
| anything else     | `task/{{issueNumber}}`                | `task/42`           |

For `feat`: convert the issue title to kebab-case (lowercase, spaces → `-`, remove special chars, max 50 chars).

---

## Step 4 — Create and switch to the branch

Use `gh issue develop` to create the branch AND link it to the issue in the Development section. If there are uncommitted local changes, stash them first.

```bash
# Stash if there are uncommitted changes
git stash

# Create branch from the issue — this establishes the Development section link
gh issue develop {{issueNumber}} \
  --repo {{owner}}/{{repo}} \
  --name {{branch-name}} \
  --base {{base-branch}} \
  --checkout

# Restore stashed changes (if any were stashed)
git stash pop
```

> **Why `gh issue develop` instead of `git checkout -b`:** GitHub only auto-links PRs to issues in the Development section when the PR targets the default branch. For feature-branch targets, the only programmatic way to establish the Development link is to create the branch via `gh issue develop`, which registers the branch-to-issue association on GitHub.

**After running `gh issue develop`, verify the link was established:**

```bash
gh issue develop {{issueNumber}} \
  --repo {{owner}}/{{repo}} \
  --list 2>/dev/null
```

If the output contains `{{branch-name}}`, the link is confirmed. If the command returns nothing or errors, **stop and warn the user** — the Development section link will be missing and must be established manually before creating the PR.

Confirm to the user: branch `{{branch-name}}` created from `{{base-branch}}` and linked to issue #{{issueNumber}}.

---

## Step 5 — Stage and commit pending changes

Check for uncommitted changes:

```bash
git status --short
```

**If there are no changes:** inform the user the branch is ready and skip to Step 7.

**If there are changes:** ask the user which files to include:

> "There are uncommitted changes. Which should I include in the first commit?
> Reply with **all** to stage everything, or list specific paths."

Wait for user reply, then stage accordingly:
- `all` → `git add -A`
- specific paths → `git add <path1> <path2> ...`

---

## Step 6 — Commit

Write the commit message following this format **strictly**:

```
{{icon}} {{type}}({{scope}}): {{short description in English, imperative mood, max 72 chars}}

{{optional body: explain WHY if non-obvious, not WHAT}}

AFFECT TASK: #{{issueNumber}}
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

**Icon map:**

| Type      | Icon |
|-----------|------|
| feat      | ✨   |
| fix       | 🐛   |
| refactor  | 🔄   |
| chore     | 🔧   |
| docs      | 📝   |
| test      | 🧪   |
| perf      | ⚡   |
| build     | 🏗️   |
| ci        | 👷   |
| style     | 💄   |
| security  | 🔒   |

**Rules:**
- Subject line: imperative mood, lowercase, no trailing period, ≤72 chars
- Body: only if "why" is non-obvious
- `AFFECT TASK` footer: always required
- Language: **English only**

Run:
```bash
git commit -m "{{full commit message}}"
```

---

## Step 7 — Push the branch

```bash
git push -u origin {{branch-name}}
```

---

## Step 8 — Create Draft Pull Request

Build the PR body using the template below:

```markdown
## Summary

{{2-4 bullet points describing what was implemented, referencing the issue}}

## Motivation

{{Why this change was needed — link to the issue or describe the problem}}

## Test plan

- [ ] {{manual or automated test steps}}

## Related

Closes #{{issueNumber}}

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

Run:
```bash
gh pr create \
  --repo {{owner}}/{{repo}} \
  --base {{base-branch}} \
  --head {{branch-name}} \
  --title "{{icon}} {{type}}({{scope}}): {{title}}" \
  --body "{{pr-body}}" \
  --draft \
  --assignee "{{gh-username}}" \
  --label "{{label}}"
```

**If the user selected a project (not "None"):** add the PR to the project after creation:

```bash
gh project item-add {{project-number}} \
  --owner "{{project-owner}}" \
  --url "{{pr-url}}"
```

Then fetch the PR item node ID:

```bash
gh project item-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json \
  --jq '.items[] | select(.content.number == {{pr-number}}) | .id'
```

Store as `prItemId`.

If the project was also used in `/create-task`, fetch its field IDs to set Type and Priority on the PR card:

```bash
gh project field-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json
```

Look for fields named `Type` and `Priority`. If found, fetch the project node ID:

```bash
gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    user(login: $owner) { projectV2(number: $number) { id } }
  }' -f owner="{{project-owner}}" -F number={{project-number}} \
  --jq '.data.user.projectV2.id' 2>/dev/null \
|| gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    organization(login: $owner) { projectV2(number: $number) { id } }
  }' -f owner="{{project-owner}}" -F number={{project-number}} \
  --jq '.data.organization.projectV2.id'
```

**Ask the user for project field values on the PR card** using `AskUserQuestion` with up to 2 questions (only for fields that exist):

**IMPORTANT — `AskUserQuestion` enforces a hard limit of 4 options per question.**

- If `projectTypeFieldId` is found: ask **"Type (PR card on project board)"** as single-select. **MANDATORY — no "None / Skip".** List actual option names only (e.g. Bug, Feature, Task). Max 4 options.
- If `projectPriorityFieldId` is found: ask **"Priority (PR card on project board)"** as single-select. Include "None / Skip" as first option, then up to 3 actual priority option names (max 4 total).

Set the fields:

```bash
# Set Type field (always — mandatory)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{prItemId}}" \
  --field-id "{{projectTypeFieldId}}" \
  --single-select-option-id "{{selectedTypeOptionId}}"

# Set Priority field (if selectedPriorityOptionId is not null)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{prItemId}}" \
  --field-id "{{projectPriorityFieldId}}" \
  --single-select-option-id "{{selectedPriorityOptionId}}"
```

Log each result: `✓ Set Type → {{name}}` / `✓ Set Priority → {{name}}`.

**Development section link — mandatory verification after PR creation:**

```bash
LINKED=$(gh issue develop {{issueNumber}} \
  --repo {{owner}}/{{repo}} \
  --list 2>/dev/null | grep "{{branch-name}}")

if [ -n "$LINKED" ]; then
  echo "✓ Development section link confirmed: branch {{branch-name}} → issue #{{issueNumber}}"
else
  echo "⚠️  Development link not detected — attempting to re-establish..."
  gh issue develop {{issueNumber}} \
    --repo {{owner}}/{{repo}} \
    --name {{branch-name}} 2>/dev/null \
    && echo "✓ Development link re-established" \
    || echo "⚠️  Could not auto-link. Open the PR sidebar → Development → Link an issue → select #{{issueNumber}}"
fi
```

---

## Step 9 — Save task metadata

After the PR is created, persist all collected resource identifiers to a JSON file so other skills (e.g. `ship-task`) can read them without re-querying GitHub.

**File path:** `~/.claude/tasks/{{repo}}/{{branch-name}}.json`
- On Windows this resolves to `C:\Users\<user>\.claude\tasks\{{repo}}\{{branch-name}}.json`
- Create parent directories if they don't exist.

**File content** (write with the Write tool — do NOT use shell redirection):

```json
{
  "username": "{{gh-username}}",
  "issueNumber": {{issueNumber}},
  "issueUrl": "https://github.com/{{owner}}/{{repo}}/issues/{{issueNumber}}",
  "branch": "{{branch-name}}",
  "baseBranch": "{{base-branch}}",
  "prUrl": "{{pr-url}}",
  "prNumber": {{pr-number}},
  "title": "{{title}}",
  "type": "{{type}}",
  "label": "{{label}}",
  "repo": "{{repo}}",
  "owner": "{{owner}}",
  "project": "{{project-name or null}}",
  "projectNumber": {{project-number or null}},
  "priority": "{{priority or null}}",
  "createdAt": "{{ISO-8601 timestamp}}"
}
```

Extract `prNumber` from the PR URL (e.g. `.../pull/43` → `43`). Use `null` (JSON null, not the string `"null"`) for optional fields that were skipped.

---

## Step 10 — Final summary

Print a summary table to the user:

```
✅ Task started successfully
─────────────────────────────────────────
Issue     #{{issueNumber}} — {{title}}
Branch    {{branch-name}} (from {{base-branch}})
PR        {{pr-url}} [DRAFT]
Project   {{project-name or "—"}}
─────────────────────────────────────────
Next: implement the changes on branch {{branch-name}}, then run /ship-task when ready for review.
```

---

## Notes

- **This skill works in any GitHub repository.** It auto-detects owner, repo, and available branches/projects.
- If `gh` is not authenticated, stop at Step 1 and tell the user to run `gh auth login`.
- Never force-push or modify history.
- The PostToolUse hook may auto-assign the PR — that is expected behaviour; do not re-assign to avoid duplicates.
