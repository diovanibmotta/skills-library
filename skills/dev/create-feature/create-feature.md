---
description: Create a GitHub feature issue spanning multiple repositories. Takes an implementation plan as context, creates a parent feature issue, then creates linked sub-tasks in each affected repository.
---

# Workflow: Create Feature

Creates a cross-repository feature: one parent issue representing the feature, plus linked sub-issues in each affected repository derived from the implementation plan.

To start working on a specific sub-task, run `/start-task <issue-number>` on the target repo afterwards.

---

## Step 0 — Detect images in the conversation

**Before anything else**, scan the current conversation for images (screenshots, attachments). Images appear as image content blocks in the conversation (not text).

- If images found: save each to temp file (e.g., `C:\Users\diova\AppData\Local\Temp\feature-img-{N}.png` or `/tmp/feature-img-{N}.png`). Store paths as `pendingImages`. Set `hasImages = true`.
- If no images: set `hasImages = false`, `pendingImages = []`.

---

## Step 1 — Detect repository context and org repos

Run in parallel:

```bash
gh repo view --json owner,name,defaultBranchRef
git branch --show-current
gh api user --jq '.login'
```

Then in parallel (using `owner` and `repo` resolved above):

```bash
gh project list --owner "@me" --format json 2>/dev/null
gh project list --owner "{{owner}}" --format json 2>/dev/null
gh repo list {{owner}} --json name,url --jq '.[] | "\(.name)"' 2>/dev/null | head -20
gh api repos/{{owner}}/{{repo}}/issue-types 2>/dev/null
```

**IMPORTANT:** These commands MUST actually be executed. Do not skip or simulate their output.

**Check token scopes** from `gh auth status`:
- `hasProjectReadScope` = `true` if scopes contain `read:project` **or** `project`
- `hasProjectWriteScope` = `true` if scopes contain `project`
- If `hasProjectReadScope = false`: warn: > ⚠️ Token missing `read:project` — project list will be empty. Fix: `gh auth refresh -s project`

From the project list JSON, extract each project's `number`, `title`, `url`. Deduplicate by number. Use actual command output — do NOT override with `[]` based on scope detection alone.

From the repo list, build `orgRepos` — list of repo names in the org (deduplicated, sorted).

From the issue types response, extract `{ id, name }` for each type (e.g. Bug, Feature, Task). Store as `repoIssueTypes`. If 404 or empty, store `null`.

Store: `owner`, `repo` (current), `defaultBranch`, `currentBranch`, `ghUsername`, list of projects (number + title), `orgRepos`, `repoIssueTypes`.

---

## Step 2 — Parse implementation plan and derive feature title

### 2a — Extract plan from arguments or request it

**If `$ARGUMENTS` is provided and contains meaningful content**: treat it as the implementation plan/context. Store as `implementationPlan`.

**If `$ARGUMENTS` is empty or too short (< 20 chars)**: ask the user in plain text:

> "Paste the implementation plan or describe the feature. Include which repositories are affected and what needs to be done in each."

Wait for reply. Store as `implementationPlan`.

### 2b — Derive feature title

**Never ask the user for the title.** Derive it from `implementationPlan`:

Analyze `implementationPlan` and distill a concise imperative title (max 72 chars, English, no trailing period) that captures the feature's main goal. E.g. `"add real-time notifications to the platform"`.

Show as a one-line note:

> "Feature title derived: **{{derived-title}}**"

### 2c — Identify affected repositories and per-repo tasks

Analyze `implementationPlan` to extract:

1. **Which repositories** are affected (match against `orgRepos` to find known repos; also accept any explicitly mentioned)
2. **What needs to be done in each repo** — a brief task description (1–2 sentences) specific to that repository's scope

Store as `repoTasks`: an array of objects `{ repoName, taskDescription }`.

Show a preview to the user (plain text, not a question):

```
Repositories identified from plan:
{{for each repoTask}}
  • {{repoName}}: {{taskDescription}}
{{/for}}
```

If no repositories can be identified from the plan, note this and proceed — the user will confirm/add repos in Step 3.

---

## Step 3 — Confirm scope and gather metadata

Ask via `AskUserQuestion` with up to 4 questions:

**IMPORTANT — `AskUserQuestion` hard limit: 4 options per question.**

1. **Feature home repository** (single-select, max 4 options — where the parent feature issue lives):
   - `{{current repo}}` (current — recommended)
   - Up to 3 other repos from `orgRepos` that seem most relevant to the feature

2. **Priority** (single-select, optional, max 4 options):
   - `None / Skip` (recommended)
   - `P1 — High`
   - `P2 — Medium`
   - `P3 — Low`

   P0 — Critical available via auto-generated "Other".

3. **GitHub Project** (single-select, optional, max 4 options — list up to 3 projects; always include "None / Skip" first):
   - `None / Skip`
   - `{{project-title-1}}` (number: {{N}})
   - `{{project-title-2}}` (number: {{N}})
   - `{{project-title-3}}` (number: {{N}})
   - If no projects found, show only "None / Skip" with description _(no projects found — if you have active projects, run `gh auth refresh -s project` and retry)_. **Do NOT add an "N/A" option.**

4. **Sub-task repositories** (multi-select if possible, otherwise single-select — list up to 4 repos from `repoTasks`; if more than 4, show the most relevant and note others can be added via `/create-subtask`):
   - For each repo in `repoTasks`: `{{repoName}}`
   - Always include current repo if not already in list

   > If `AskUserQuestion` does not support multi-select, ask as single-select with "All identified repos" as the first option meaning proceed with all `repoTasks`.

Store: `homeRepo` (owner/name of parent issue), `priority`, `selectedProject`, `confirmedRepos` (list of repo names to create sub-tasks in).

---

## Step 4 — Fetch project fields (if project selected)

**If the user selected a project (not "None / Skip")**, fetch custom fields:

```bash
gh project field-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json
```

Parse JSON. Look for `Type` and `Priority` fields (case-insensitive). Extract `field.id` and `field.options[]` (`{ id, name }`).

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

**Ask for project-specific field values** via `AskUserQuestion` with up to 2 questions:

- If `projectTypeFieldId` found: ask **"Type (on project board)"** — single-select, **MANDATORY (no "None / Skip")**. Always pre-select the option matching "Feature" (case-insensitive). Max 4 options.
- If `projectPriorityFieldId` found: ask **"Priority (on project board)"** — single-select. "None / Skip" first, then up to 3 actual priority names. Pre-select option matching priority from Step 3. Max 4 options.

Store `selectedTypeOptionId` and `selectedPriorityOptionId` (null if skipped).

Show: `Board fields: Type → **{{name or "—"}}** | Priority → **{{name or "—"}}**`

If neither field found, skip silently.

---

## Step 5 — Resolve GitHub issue type

Map `feat` → GitHub issue type:

- If `repoIssueTypes` not null/empty: match option whose name contains "Feature" (case-insensitive). If no match, use first available option as fallback.
- Store as `ghIssueTypeName`. Show: `Issue type: **{{ghIssueTypeName}}**`
- If `repoIssueTypes` is null: store `null`, skip silently.

---

## Step 6 — Create parent feature issue

Build body for the parent feature issue:

```markdown
## Overview

{{implementationPlan}}

## Type

Feature

{{#if priority is not "None / Skip"}}
## Priority

{{priority value}}
{{/if}}

## Affected Repositories

{{for each confirmedRepo}}
- [ ] {{repoName}}: {{taskDescription}}
{{/for}}

{{#if hasImages}}
## Screenshots

{{for each image: ![screenshot-N]({{placeholder}})}}
{{/if}}
```

Ensure label exists in `homeRepo`:

```bash
gh label create "enhancement" \
  --repo {{homeOwner}}/{{homeRepo}} \
  --color "#a2eeef" \
  --force
```

Create the parent feature issue:

```bash
gh issue create \
  --repo {{homeOwner}}/{{homeRepo}} \
  --title "{{derived-title}}" \
  --body "{{body}}" \
  --assignee "{{ghUsername}}" \
  --label "enhancement" \
  {{#if ghIssueTypeName}}--type "{{ghIssueTypeName}}"{{/if}}
```

> If `--type` fails (older CLI), retry without it and warn the user.

Store `featureIssueNumber` from output URL. Store `featureIssueUrl` = `https://github.com/{{homeOwner}}/{{homeRepo}}/issues/{{featureIssueNumber}}`.

---

### Step 6a — Upload images (if hasImages is true)

**Run only if `hasImages = true`.**

**6a-1: Ensure `issue-assets` release exists in `homeRepo`:**

```bash
gh release view issue-assets --repo {{homeOwner}}/{{homeRepo}} > /dev/null 2>&1 \
  || gh release create issue-assets \
       --repo {{homeOwner}}/{{homeRepo}} \
       --title "Issue Assets" \
       --notes "Automated image storage for GitHub issues. Do not delete." \
       --prerelease
```

**6a-2: Upload each image:**

```bash
gh release upload issue-assets "feature-{{featureIssueNumber}}-img-{{N}}.png" \
  --repo {{homeOwner}}/{{homeRepo}} \
  --clobber
```

**6a-3: Fetch URLs and update issue body:**

```bash
gh release view issue-assets \
  --repo {{homeOwner}}/{{homeRepo}} \
  --json assets \
  --jq '.assets[] | select(.name | startswith("feature-{{featureIssueNumber}}-")) | .browser_download_url'
```

Update issue:

```bash
gh issue edit {{featureIssueNumber}} \
  --repo {{homeOwner}}/{{homeRepo}} \
  --body "{{updated-body-with-real-image-urls}}"
```

On failure: add `> ⚠️ Screenshots could not be uploaded automatically. Please attach them manually.` to issue body.

---

## Step 7 — Create sub-tasks in each affected repository

For **each repo in `confirmedRepos`**, execute the following sequence. Process repos **sequentially** (not in parallel) to avoid rate limits and to show progress to the user.

Log before each: `Creating sub-task for {{repoName}}...`

### 7a — Resolve issue type for target repo

```bash
gh api repos/{{owner}}/{{repoName}}/issue-types 2>/dev/null
```

Map `feat` → option containing "Feature". Store as `targetGhIssueTypeName` (null if not available).

### 7b — Ensure label exists in target repo

```bash
gh label create "enhancement" \
  --repo {{owner}}/{{repoName}} \
  --color "#a2eeef" \
  --force
```

### 7c — Build sub-task body

```markdown
## Context

{{taskDescription for this repo}}

## Type

Feature

## Parent Feature

{{featureIssueUrl}}

{{#if priority is not "None / Skip"}}
## Priority

{{priority value}}
{{/if}}
```

### 7d — Create sub-task issue

```bash
gh issue create \
  --repo {{owner}}/{{repoName}} \
  --title "{{derived-title}}" \
  --body "{{subtask-body}}" \
  --assignee "{{ghUsername}}" \
  --label "enhancement" \
  {{#if targetGhIssueTypeName}}--type "{{targetGhIssueTypeName}}"{{/if}}
```

Store `subIssueNumber`. Store `subIssueUrl`.

### 7e — Link sub-task to parent feature

Fetch child issue database ID:

```bash
gh api repos/{{owner}}/{{repoName}}/issues/{{subIssueNumber}} \
  --jq '.id'
```

Store as `childIssueId`.

Link as sub-issue of parent feature:

```bash
gh api -X POST \
  repos/{{homeOwner}}/{{homeRepo}}/issues/{{featureIssueNumber}}/sub_issues \
  --field sub_issue_id={{childIssueId}}
```

If success: log `✓ Linked {{repoName}}#{{subIssueNumber}} → feature#{{featureIssueNumber}}`

If fails (sub-issues not available): warn:
> "⚠️ Could not auto-link {{repoName}}#{{subIssueNumber}}. Open feature#{{featureIssueNumber}} and add manually."

Store result in `createdSubtasks`: `{ repoName, issueNumber, issueUrl, linked }`.

---

## Step 8 — Add parent feature to project board (if project selected)

```bash
gh project item-add {{project-number}} \
  --owner "{{project-owner}}" \
  --url "{{featureIssueUrl}}"
```

Fetch item node ID:

```bash
gh project item-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json \
  --jq '.items[] | select(.content.number == {{featureIssueNumber}}) | .id'
```

Store as `featureItemId`.

Set Type and Priority fields:

```bash
# Type (always — mandatory)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{featureItemId}}" \
  --field-id "{{projectTypeFieldId}}" \
  --single-select-option-id "{{selectedTypeOptionId}}"

# Priority (if selectedPriorityOptionId is not null)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{featureItemId}}" \
  --field-id "{{projectPriorityFieldId}}" \
  --single-select-option-id "{{selectedPriorityOptionId}}"
```

Log: `✓ Set Type → Feature` / `✓ Set Priority → {{name}}`.

---

## Step 9 — Final summary

```
✅ Feature created successfully
─────────────────────────────────────────
Feature   {{homeOwner}}/{{homeRepo}}#{{featureIssueNumber}} — {{derived-title}}
Type      Feature
Priority  {{priority or "—"}}
Project   {{project-name or "—"}}

Sub-tasks created:
{{for each createdSubtask}}
  • {{repoName}}#{{issueNumber}} — {{linked ? "✓ linked" : "⚠️ manual link required"}}
    {{issueUrl}}
{{/for}}
─────────────────────────────────────────
To start a sub-task: /start-task <issue-number> (in the target repo)
```

---

## Notes

- **Always type = Feature.** No type question is asked — every issue created by this skill is a feature.
- Sub-tasks all receive the same title as the parent feature; per-repo context lives in the body.
- If `confirmedRepos` includes `homeRepo`, a sub-task is created there too (in addition to the parent feature issue).
- Sub-issues API depends on GitHub plan/org settings. If unavailable, the skill falls back to manual linking instructions.
- If `gh` is not authenticated, stop at Step 1 and tell the user to run `gh auth login`.
- If a repo in `confirmedRepos` does not exist or is inaccessible, log a warning, skip it, and continue with the rest.
- The PostToolUse hook may auto-assign issues — expected behaviour; do not re-assign to avoid duplicates.
- For features with more than 4 repos, only the first 4 are shown in Step 3 question. The rest are still processed from `repoTasks`; inform the user additional repos beyond the 4 shown will also receive sub-tasks if identified from the plan.
