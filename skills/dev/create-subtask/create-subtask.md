---
description: Create a GitHub sub-issue linked to a parent issue. The sub-issue can be created in any repository. Works like /create-task but always attaches to a parent via GitHub's sub-issue feature.
---

# Workflow: Create Subtask

Creates a GitHub issue in any repository and links it as a sub-issue of a parent issue. Works in **any** GitHub repository.

To start working on the sub-issue (branch + PR), run `/start-task <issue-number>` after this.

---

## Step 1 — Detect repository context

Run the following in parallel:

```bash
gh repo view --json owner,name,defaultBranchRef
git branch --show-current
git branch -a --format="%(refname:short)" | sed 's|origin/||' | sort -u | head -40
```

Also fetch projects and issue types in parallel:

```bash
gh project list --owner "@me" --format json 2>/dev/null
gh project list --owner "<org>" --format json 2>/dev/null
gh api repos/{{currentOwner}}/{{currentRepo}}/issue-types 2>/dev/null
```

From the project list JSON, extract each project's `number`, `title`, and `url`. Deduplicate by number.

From the issue types response, extract `{ id, name }` for each available type (e.g. Bug, Feature, Task). Store as `repoIssueTypes`. If the endpoint returns 404 or empty, store `null`.

Store: `currentOwner`, `currentRepo`, `defaultBranch`, `currentBranch`, available branches, list of projects (number + title), `repoIssueTypes`.

---

## Step 2 — Resolve parent issue

**Ask the user for the parent issue** in plain text (free-form message — NOT AskUserQuestion):

> "Which issue is the parent? Enter as `owner/repo#number` (e.g. `fluxup-platform/fluxup-bff#42`) or just `#number` if it's in the current repository."

Parse the reply:
- If format is `owner/repo#number`: store `parentOwner`, `parentRepo`, `parentNumber`.
- If format is `#number` or just a number: use `currentOwner`/`currentRepo` as parent owner/repo, store `parentNumber`.

Fetch parent issue details to confirm it exists and show a one-line note:

```bash
gh issue view {{parentNumber}} \
  --repo {{parentOwner}}/{{parentRepo}} \
  --json number,title \
  --jq '"#\(.number) — \(.title)"'
```

Show: `Parent issue: **#{{parentNumber}} — {{parentTitle}}**`

---

## Step 3 — Derive title and gather task information

### 3a — Derive the title automatically

**Never ask the user for the title.** Derive it from available context in this priority order:

1. **Skill arguments** (`$ARGUMENTS`): if the user passed text when invoking the skill, distill it into a concise imperative title (max 72 chars, English, no trailing period).
2. **Recent git commits** (`git log -5 --oneline`): if no arguments were given, infer the title from the most recent commit messages on the current branch.
3. **Current branch name**: humanize the branch name (e.g. `feature/user-auth` → `"add user authentication"`).
4. **Fallback**: use `"work in progress"` and note it to the user.

Store the derived title. Show it as a one-line note:

> "Title derived from context: **{{derived-title}}**"

### 3b — Ask for type, target repository, base branch, priority, and project

**MANDATORY:** Always ask all 4 questions below via `AskUserQuestion`. Never infer or skip the task type question.

**IMPORTANT — `AskUserQuestion` enforces a hard limit of 4 options per question.** Never exceed this limit.

1. **Task type** (single-select, **max 4 options**):
   - `feat` — New feature (branch: `feature/<kebab-name>`)
   - `fix` — Bug fix (branch: `bugfix/<issue-number>`)
   - `refactor` — Refactor (branch: `task/<issue-number>`)
   - `chore` — Chore / config / deps / docs / test / perf / build / ci / style (branch: `task/<issue-number>`)

   If the user selects "Other" (auto-generated), treat it as free text. Map it to the closest semantic type from: `chore`, `docs`, `test`, `perf`, `build`, `ci`, `style`.

2. **Target repository** (single-select, **max 4 options** — list repos available in the org; always include the current repo as first option with "(current)" label; list up to 3 others):

   To fetch available repos in the org:
   ```bash
   gh repo list {{currentOwner}} --json name --jq '.[].name' | head -10
   ```
   Show up to 3 other repos besides the current one. If only 1 repo exists, pre-select it silently and skip this question (replace with another relevant question or collapse to 3 questions).

3. **Priority** (single-select, **optional**, **max 4 options**):
   - `None / Skip` (recommended — priority can be set later)
   - `P1 — High`
   - `P2 — Medium`
   - `P3 — Low`

   P0 — Critical is available via the auto-generated "Other" option.

4. **GitHub Project** (single-select, **optional**, **max 4 options** — list up to 3 projects; always include "None / Skip" as first option):
   - `None / Skip`
   - `{{project-title-1}}` (number: {{N}})
   - `{{project-title-2}}` (number: {{N}})
   - `{{project-title-3}}` (number: {{N}}) *(include only if ≤3 projects found)*
   - If no projects were found, show only "None / Skip"

Store the selected target repo as `targetOwner`/`targetRepo` (resolve owner from `currentOwner` if only repo name was listed).

### 3c — Fetch project fields (if project selected)

**If the user selected a project (not "None / Skip")**, fetch that project's custom fields:

```bash
gh project field-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json
```

Parse the JSON. Look for fields named `Type` and `Priority` (case-insensitive). Extract `field.id` and `field.options[]` (`{ id, name }`).

Store: `projectTypeFieldId`, `projectTypeOptions`, `projectPriorityFieldId`, `projectPriorityOptions`.

Fetch project node ID:

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

Store as `projectNodeId`.

**Ask the user for project field values** using `AskUserQuestion` with up to 2 questions:

**IMPORTANT — `AskUserQuestion` hard limit: 4 options per question.**

- If `projectTypeFieldId` is found: ask **"Type (on project board)"** as single-select. **MANDATORY — no "None / Skip".** List actual option names only (e.g. Bug, Feature, Task). Max 4 options.
- If `projectPriorityFieldId` is found: ask **"Priority (on project board)"** as single-select. Include "None / Skip" as first option, then up to 3 actual priority option names (max 4 total).

If neither field found, skip silently.

Store `selectedTypeOptionId` and `selectedPriorityOptionId` (null if skipped).

Show confirmation: `Board fields: Type → **{{name or "—"}}** | Priority → **{{name or "—"}}**`

### 3d — Optional description

Ask the user in plain text:

> "Optionally add a description for this subtask (or press Enter to skip)."

Use the reply as issue body description. If skipped, use the derived title. Do **not** change the title.

---

## Step 4 — Resolve GitHub issue type

**Before creating the issue**, resolve the native GitHub issue type from `repoIssueTypes`:

- If `repoIssueTypes` is not null/empty, map the task type chosen in Step 3b to a GitHub issue type:
  - `feat` → match option whose name contains "Feature" (case-insensitive)
  - `fix` → match option whose name contains "Bug"
  - `refactor`, `chore`, `docs`, `test`, `perf`, `build`, `ci`, `style` → match option whose name contains "Task"
  - If no match found for the mapped name, use the first available option as fallback.
- Store the matched type name as `ghIssueTypeName`. Show a one-line note: `Issue type: **{{ghIssueTypeName}}**` (auto-mapped — no question needed).
- If `repoIssueTypes` is null, store `ghIssueTypeName` as `null` and skip silently.

---

## Step 5 — Create GitHub issue in target repository

Build the issue body:

```markdown
## Context

{{user-provided description, or "No description provided." if omitted}}

## Type

{{task type label: Feature / Bug Fix / Refactor / Chore / Docs / Tests / Performance}}

## Parent Issue

{{parentOwner}}/{{parentRepo}}#{{parentNumber}}

{{#if priority is not "None / Skip"}}
## Priority

{{priority value}}
{{/if}}
```

Get current GitHub username:
```bash
gh api user --jq '.login'
```

Ensure the label exists in the **target** repository (create if missing):
```bash
gh label create "{{label}}" \
  --repo {{targetOwner}}/{{targetRepo}} \
  --color "{{color}}" \
  --force
```

Create the issue:
```bash
gh issue create \
  --repo {{targetOwner}}/{{targetRepo}} \
  --title "{{title}}" \
  --body "{{body}}" \
  --assignee "{{gh-username}}" \
  --label "{{label}}" \
  {{#if ghIssueTypeName}}--type "{{ghIssueTypeName}}"{{/if}}
```

> If `gh` returns an error for `--type` (older CLI version), retry without the flag and warn the user to set the type manually.

**Label mapping:**

| Type     | Label name    | Color    |
|----------|---------------|----------|
| feat     | `enhancement` | `#a2eeef` |
| fix      | `bug`         | `#d73a4a` |
| refactor | `refactor`    | `#e4e669` |
| chore    | `chore`       | `#cfd3d7` |
| docs     | `documentation` | `#0075ca` |
| test     | `test`        | `#bfd4f2` |
| perf     | `performance` | `#0e8a16` |
| build    | `build`       | `#e99695` |
| ci       | `ci`          | `#f9d0c4` |
| style    | `style`       | `#fef2c0` |

Store `issueNumber` from output (e.g. `https://github.com/owner/repo/issues/42` → `42`).

---

## Step 6 — Link as sub-issue of parent

Fetch the child issue's database ID (integer, not issue number):

```bash
gh api repos/{{targetOwner}}/{{targetRepo}}/issues/{{issueNumber}} \
  --jq '.id'
```

Store as `childIssueId`.

Link the child issue as a sub-issue of the parent:

```bash
gh api -X POST \
  repos/{{parentOwner}}/{{parentRepo}}/issues/{{parentNumber}}/sub_issues \
  --field sub_issue_id={{childIssueId}}
```

If the command succeeds, log: `✓ Linked as sub-issue of #{{parentNumber}}`.

If it fails (e.g. sub-issues not enabled on the plan), warn the user:

> "⚠️ Could not link automatically. Open issue #{{parentNumber}} and click 'Create sub-issue' → 'Add existing issue' → paste the URL of issue #{{issueNumber}}."

---

## Step 7 — Add to project board (if project selected)

```bash
gh project item-add {{project-number}} \
  --owner "{{project-owner}}" \
  --url "https://github.com/{{targetOwner}}/{{targetRepo}}/issues/{{issueNumber}}"
```

Fetch item node ID:

```bash
gh project item-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json \
  --jq '.items[] | select(.content.number == {{issueNumber}}) | .id'
```

Store as `issueItemId`.

Set Type and Priority fields:

```bash
# Set Type field (always — mandatory)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{issueItemId}}" \
  --field-id "{{projectTypeFieldId}}" \
  --single-select-option-id "{{selectedTypeOptionId}}"

# Set Priority field (if selectedPriorityOptionId is not null)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{issueItemId}}" \
  --field-id "{{projectPriorityFieldId}}" \
  --single-select-option-id "{{selectedPriorityOptionId}}"
```

Log: `✓ Set Type → {{name}}` / `✓ Set Priority → {{name}}`.

---

## Step 8 — Final summary

```
✅ Subtask created successfully
─────────────────────────────────────────
Issue     {{targetOwner}}/{{targetRepo}}#{{issueNumber}} — {{title}}
Parent    {{parentOwner}}/{{parentRepo}}#{{parentNumber}} — {{parentTitle}}
Sub-link  {{linked or "⚠️ manual linking required"}}
Type      {{semantic-type}} | Board: {{board-type-name or "—"}}
Priority  {{priority or "—"}}
Project   {{project-name or "—"}}
─────────────────────────────────────────
Ready to start? Run: /start-task {{issueNumber}}
```

---

## Notes

- Sub-issues are a GitHub feature — availability depends on the plan/org settings. If the API returns 404 or 422, fall back to the manual linking warning in Step 5.
- The subtask issue is created in `targetRepo`, but the sub-issue link is registered on the `parentRepo` issue. Both can be in different repos.
- If `gh` is not authenticated, stop at Step 1 and tell the user to run `gh auth login`.
- Never force-push or modify history.
- The PostToolUse hook may auto-assign the issue — that is expected behaviour; do not re-assign.
