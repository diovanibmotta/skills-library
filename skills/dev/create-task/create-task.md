---
description: Create a GitHub issue and add it to the project board. Works in any GitHub repository. Use /start-task <issue-number> to create the branch and PR afterwards.
---

# Workflow: Create Task

Creates a GitHub issue and adds it to the project board. Works in **any** GitHub repository.

To start working on the issue (branch + PR), run `/start-task <issue-number>` after this.

---

## Step 0 — Detect images in the conversation

**Before anything else**, scan the current conversation for images sent by the user (screenshots, attachments, photos). Images appear as image content blocks in the conversation (not text).

- If images are found: save each one to a temp file (e.g., `C:\Users\diova\AppData\Local\Temp\issue-img-{N}.png` or `/tmp/issue-img-{N}.png`). Store the list of temp file paths as `pendingImages`. Set `hasImages = true`.
- If no images found: set `hasImages = false`, `pendingImages = []`.

Do **not** skip this step. Image detection must happen before any `gh` commands run.

---

## Step 1 — Detect repository context

Run the following in parallel:

```bash
gh repo view --json owner,name,defaultBranchRef
git branch --show-current
git branch -a --format="%(refname:short)" | sed 's|origin/||' | sort -u | head -40
```

Then fetch the org/user name from the repo and list projects and issue types in parallel:

```bash
# Replace <org> with the owner value obtained above
gh project list --owner "@me" --format json 2>/dev/null
gh project list --owner "<org>" --format json 2>/dev/null
gh api repos/{{owner}}/{{repo}}/issue-types 2>/dev/null
```

From the project list JSON, extract each project's `number`, `title`, and `url`. Deduplicate by number.

From the issue types response, extract `{ id, name }` for each available type (e.g. Bug, Feature, Task). Store as `repoIssueTypes`. If the endpoint returns 404 or empty, store `null`.

Store: `owner`, `repo`, `defaultBranch`, `currentBranch`, available branches, list of projects (number + title), `repoIssueTypes`.

---

## Step 2 — Derive title and gather task information

### 2a — Derive the title automatically

**Never ask the user for the title.** Derive it from available context in this priority order:

1. **Skill arguments** (`$ARGUMENTS`): if the user passed text when invoking the skill, distill it into a concise imperative title (max 72 chars, English, no trailing period). E.g. `"add weekly view navigation to calendar"`.
2. **Recent git commits** (`git log -5 --oneline`): if no arguments were given, infer the title from the most recent commit messages on the current branch that describe in-progress work.
3. **Current branch name**: if neither above yields a clear title, humanize the branch name (e.g. `feature/user-auth` → `"add user authentication"`).
4. **Fallback**: if no context is available, generate a generic title like `"work in progress"` and note it to the user.

Store the derived title. Show it to the user as a one-line note before the questions (not a question — just informational):

> "Title derived from context: **{{derived-title}}**"

### 2b — Ask for type, base branch, priority, and project

**MANDATORY:** Always ask all 4 questions below via `AskUserQuestion`. Never infer or skip the task type question — it must always be explicitly confirmed by the user.

Ask the user the following questions **all at once** using `AskUserQuestion` with **4 questions**.

**IMPORTANT — `AskUserQuestion` enforces a hard limit of 4 options per question.** Never exceed this limit. Group rare types together and rely on the auto-generated "Other" option for edge cases.

1. **Task type** (single-select, **max 4 options**):
   - `feat` — New feature (branch: `feature/<kebab-name>`)
   - `fix` — Bug fix (branch: `bugfix/<issue-number>`)
   - `refactor` — Refactor (branch: `task/<issue-number>`)
   - `chore` — Chore / config / deps / docs / test / perf / build / ci / style (branch: `task/<issue-number>`)

   If the user selects "Other" (auto-generated), treat it as the type they type in free text. Map it to the closest semantic type from: `chore`, `docs`, `test`, `perf`, `build`, `ci`, `style`.

2. **Base branch** (single-select, **max 4 options** — list up to 3 detected branches from Step 1, mark `main` or `develop` as recommended if present)

3. **Priority** (single-select, **optional**, **max 4 options** — include "None / Skip" as first option):
   - `None / Skip` (recommended — priority can be set later)
   - `P1 — High`
   - `P2 — Medium`
   - `P3 — Low`

   P0 — Critical is available via the auto-generated "Other" option.

4. **GitHub Project** (single-select, **optional**, **max 4 options** — list up to 3 projects detected in Step 1 by title; always include "None / Skip" as first option):
   - `None / Skip`
   - `{{project-title-1}}` (number: {{N}})
   - `{{project-title-2}}` (number: {{N}})
   - `{{project-title-3}}` (number: {{N}}) *(include only if ≤3 projects found)*
   - If no projects were found, show only "None / Skip"

### 2c — Fetch project fields (if project selected)

**If the user selected a project (not "None / Skip")**, immediately fetch that project's custom fields to discover available Type and Priority options on the board:

```bash
gh project field-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json
```

Parse the JSON response. Look for fields named `Type` and `Priority` (case-insensitive). For each found field, extract:
- `field.id` — the field node ID
- `field.options[]` — array of `{ id, name }` objects (for single-select fields)

Store:
- `projectTypeFieldId` — node ID of the Type field (or `null` if not found)
- `projectTypeOptions` — array of `{ id, name }` for Type options
- `projectPriorityFieldId` — node ID of the Priority field (or `null` if not found)
- `projectPriorityOptions` — array of `{ id, name }` for Priority options

Also fetch the project's node ID (needed for `gh project item-edit`):

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

**Then ask the user for project-specific field values** using `AskUserQuestion` with up to 2 questions (only ask questions for fields that actually exist on the project):

**IMPORTANT — `AskUserQuestion` enforces a hard limit of 4 options per question.**

- If `projectTypeFieldId` is found: ask **"Type (on project board)"** as single-select. **This field is MANDATORY — do NOT include a "None / Skip" option.** List the project's actual option names only (e.g. Bug, Feature, Task). Max 4 options. The user must pick one; this value will always be set on the card.
- If `projectPriorityFieldId` is found: ask **"Priority (on project board)"** as single-select. Include "None / Skip" as first option, then up to 3 of the project's actual priority option names (max 4 total). Pre-select the option that best matches the priority chosen in Step 2b (e.g., if user chose "P2 — Medium", highlight the option containing "Medium" or "P2").

If neither field is found on the project, skip this step silently and proceed.

Store the selected option IDs as `selectedTypeOptionId` and `selectedPriorityOptionId` (null if skipped).

Show a one-line confirmation to the user before proceeding:

> "Board fields: Type → **{{selected type name or "—"}}** | Priority → **{{selected priority name or "—"}}**"

### 2d — Mandatory description

After receiving the answers from 2b and 2c (project field selections), ask the user in plain text (free-form message — NOT AskUserQuestion):

> "Describe the problem or task in detail. What is happening? What should happen instead? Include steps to reproduce if applicable."

**This description is required.** If the user sends nothing meaningful (empty reply or single word), prompt once more:

> "A detailed description is required. Please describe the problem, expected behavior, and any relevant context."

Use the user's description as the issue body. Do **not** use this reply to change the title. Do **not** fall back to the title as the description — wait for real input.

---

## Step 3 — Resolve GitHub issue type

**Before creating the issue**, resolve the native GitHub issue type from `repoIssueTypes`:

- If `repoIssueTypes` is not null/empty, map the task type chosen in Step 2b to a GitHub issue type:
  - `feat` → match option whose name contains "Feature" (case-insensitive)
  - `fix` → match option whose name contains "Bug"
  - `refactor`, `chore`, `docs`, `test`, `perf`, `build`, `ci`, `style` → match option whose name contains "Task"
  - If no match found for the mapped name, use the first available option as fallback.
- Store the matched type name as `ghIssueTypeName`. Show a one-line note: `Issue type: **{{ghIssueTypeName}}**` (auto-mapped — no question needed).
- If `repoIssueTypes` is null, store `ghIssueTypeName` as `null` and skip silently.

---

## Step 4 — Create GitHub issue

Build the issue body using the template below. Fill every `{{placeholder}}` before running the command.

**Issue body template:**

```markdown
## Context

{{user-provided description}}

## Type

{{task type label: Feature / Bug Fix / Refactor / Chore / Docs / Tests / Performance}}

{{#if priority is not "None / Skip"}}
## Priority

{{priority value}}
{{/if}}

{{#if hasImages}}
## Screenshots

{{for each image: ![screenshot-N]({{placeholder — will be replaced after upload}})}}
{{/if}}
```

> The `{{placeholder}}` lines in the Screenshots section will be replaced with real URLs after images are uploaded (see Step 4a).

Run:

```bash
gh issue create \
  --repo {{owner}}/{{repo}} \
  --title "{{title}}" \
  --body "{{body}}" \
  --assignee "{{gh-username}}" \
  --label "{{label}}" \
  {{#if ghIssueTypeName}}--type "{{ghIssueTypeName}}"{{/if}}
```

> If `gh` returns an error for `--type` (older CLI version), retry without the flag and warn the user to set the type manually.

**Label mapping** (create label first if it doesn't exist using `gh label create`):

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

To get the current GitHub username:
```bash
gh api user --jq '.login'
```

Store the created issue number from the command output (e.g., `https://github.com/owner/repo/issues/42` → `42`).

---

### Step 4a — Upload images (if hasImages is true)

**Run only if `hasImages = true`.** Upload each image in `pendingImages` to a dedicated GitHub release used as a static asset store, then update the issue body with the real image URLs.

**4a-1: Ensure the `issue-assets` release tag exists:**

```bash
gh release view issue-assets --repo {{owner}}/{{repo}} > /dev/null 2>&1 \
  || gh release create issue-assets \
       --repo {{owner}}/{{repo}} \
       --title "Issue Assets" \
       --notes "Automated image storage for GitHub issues. Do not delete." \
       --prerelease
```

**4a-2: Upload each image and collect URLs:**

For each file in `pendingImages` (e.g., `issue-img-1.png`), rename it to include the issue number for uniqueness (`issue-{{issue-number}}-img-{{N}}.png`), then upload:

```bash
# Rename for uniqueness, then upload
gh release upload issue-assets "issue-{{issue-number}}-img-{{N}}.png" \
  --repo {{owner}}/{{repo}} \
  --clobber
```

After all uploads, fetch the download URLs:

```bash
gh release view issue-assets \
  --repo {{owner}}/{{repo}} \
  --json assets \
  --jq '.assets[] | select(.name | startswith("issue-{{issue-number}}-")) | .browser_download_url'
```

Store the list as `imageUrls`.

**4a-3: Update the issue body** with real image URLs. Replace each `{{placeholder}}` in the Screenshots section with the actual URL:

```bash
gh issue edit {{issue-number}} \
  --repo {{owner}}/{{repo}} \
  --body "{{updated-body-with-real-image-urls}}"
```

Log: `✓ Uploaded {{N}} image(s) and attached to issue #{{issue-number}}`

**If any upload step fails**, log a warning and continue — add a note in the issue body instead:
> `> ⚠️ Screenshots could not be uploaded automatically. Please attach them manually.`

---

**If the user selected a project (not "None"):**

```bash
gh project item-add {{project-number}} \
  --owner "{{project-owner}}" \
  --url "https://github.com/{{owner}}/{{repo}}/issues/{{issue-number}}"
```

Then fetch the item node ID for the newly added issue:

```bash
gh project item-list {{project-number}} \
  --owner "{{project-owner}}" \
  --format json \
  --jq '.items[] | select(.content.number == {{issue-number}}) | .id'
```

Store as `issueItemId`.

**Set Type and Priority fields on the issue project item** (if field IDs and option IDs were collected in Step 2c):

```bash
# Set Type field (always set — selectedTypeOptionId is never null at this point)
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

Run the Type command always (it is mandatory). Run Priority only if the user made a selection (not "None / Skip"). Log each result: `✓ Set Type → {{name}}` / `✓ Set Priority → {{name}}`.

---

## Step 5 — Final summary

Print a summary to the user:

```
✅ Issue created successfully
─────────────────────────────────────────
Issue     #{{issue-number}} — {{title}}
Type      {{semantic-type}} | Board: {{board-type-name or "—"}}
Priority  {{priority or "—"}}
Project   {{project-name or "—"}}
─────────────────────────────────────────
Ready to start? Run: /start-task {{issue-number}}
```

---

## Notes

- **This skill works in any GitHub repository.** It auto-detects owner, repo, and available branches/projects.
- If the repository has no projects, show only "None / Skip" in the project question and skip all project-related steps.
- If `gh` is not authenticated, stop at Step 1 and tell the user to run `gh auth login`.
- The PostToolUse hook may auto-assign the issue — that is expected behaviour; do not re-assign to avoid duplicates.
