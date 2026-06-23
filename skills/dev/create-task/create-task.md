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

## Step 1 — Detect repository context and available options

Run everything in parallel:

```bash
gh repo view --json owner,name,defaultBranchRef
git branch --show-current
git branch -a --format="%(refname:short)" | sed 's|origin/||' | sort -u | head -40
gh auth status 2>&1
```

Parse `gh auth status` output. Look for the `Token scopes:` line.

- `hasProjectReadScope` = `true` if scopes contain `read:project` **or** `project`
- `hasProjectWriteScope` = `true` if scopes contain `project`

If `hasProjectReadScope = false`: print a **non-blocking warning** and continue (do not stop):

> ⚠️ Token missing `read:project` scope — project list will be empty.
> Fix now: `gh auth refresh -s read:project` (list only) or `gh auth refresh -s project` (full board access), then re-run `/create-task`.

If `hasProjectReadScope = true` but `hasProjectWriteScope = false`: print a **non-blocking warning**:

> ⚠️ Token has `read:project` but not `project` — projects will be listed but items cannot be added to the board.
> Fix: `gh auth refresh -s project`

Store: `owner`, `repo`, `defaultBranch`, `currentBranch`, available branches.

### 1b — Fetch available options from GitHub

Run all in parallel (using `owner` and `repo` resolved in Step 1):

```bash
gh api repos/{{owner}}/{{repo}}/labels --jq '.[].name' 2>/dev/null
gh api repos/{{owner}}/{{repo}}/issue-types 2>/dev/null
gh project list --owner "@me" --format json 2>/dev/null
gh project list --owner "{{owner}}" --format json 2>/dev/null
```

**IMPORTANT:** These commands MUST actually be executed. Do not skip or simulate their output. Use the real results to populate the project list in Q4.

**Issue types** (`gh api repos/.../issue-types`): extract `{ id, name }` for each type. Store as `repoIssueTypes`. If 404/empty, store `null`.

**Projects**: combine results from both `gh project list` calls. Deduplicate by number. Extract `number`, `title`. Store as `availableProjects`. If both commands returned errors or empty results, store `[]`.

**IMPORTANT:** Do NOT set `availableProjects = []` based on scope detection alone — always use actual command output.

Show a one-line discovery summary before asking questions:

> Found: **{{N}} issue type(s)** | **{{M}} project(s)**
> _(if hasProjectReadScope = false: "Projects unavailable — token missing `read:project` scope. Run: `gh auth refresh -s project`")_

---

## Step 2 — Derive title and gather task information

### 2a — Derive the title automatically

**Never ask the user for the title.** Derive it from available context in this priority order:

1. **Skill arguments** (`$ARGUMENTS`): if the user passed text when invoking the skill, distill it into a concise imperative title (max 72 chars, English, no trailing period).
2. **Recent git commits** (`git log -5 --oneline`): infer title from most recent commits on current branch.
3. **Current branch name**: humanize the branch name (e.g. `feature/user-auth` → `"add user authentication"`).
4. **Fallback**: `"work in progress"`.

Show as one-line note before questions:

> "Title derived from context: **{{derived-title}}**"

### 2b — Ask for type, base branch, project, and GitHub issue type

**MANDATORY:** Always ask all questions below via `AskUserQuestion`. Never infer or skip.

**IMPORTANT — `AskUserQuestion` enforces a hard limit of 4 options per question.**

Ask **all at once** using `AskUserQuestion` with **4 questions**:

**Q1 — Task type** (single-select, max 4 options — semantic type for branch naming and commit convention):
- `feat` — New feature → branch `feature/<kebab-name>`
- `fix` — Bug fix → branch `bugfix/<issue-number>`
- `refactor` — Refactor → branch `task/<issue-number>`
- `chore` — Chore / config / deps / docs / test / perf / build / ci / style → branch `task/<issue-number>`

If user selects "Other" (auto-generated), treat as free-text and map to closest: `chore`, `docs`, `test`, `perf`, `build`, `ci`, `style`.

**Q2 — Base branch** (single-select, max 4 options — list up to 3 detected branches from Step 1; mark `main` or `develop` as recommended if present)

**Q3 — GitHub issue type** (single-select, max 4 options):

- **If `repoIssueTypes` is not null/empty**: list the REAL issue type names from GitHub (e.g. `Feature`, `Bug`, `Task`, `Enhancement`). Show up to 4. Include descriptions from the API response if available.
- **If `repoIssueTypes` is null**: show these fallback options:
  - `Feature` — new capability
  - `Bug` — something broken
  - `Task` — work item / chore
  - (use "Other" for anything else)

**Q4 — GitHub Project** (single-select, max 4 options — always include "None / Skip" as first option):

- **If `availableProjects` is not empty**: list up to 3 project titles. E.g.:
  - `None / Skip`
  - `{{project-title-1}}` (number: {{N}})
  - `{{project-title-2}}` (number: {{N}})
- **If `availableProjects` is empty**: show only `None / Skip` with description _(no projects found — if you have active projects, run `gh auth refresh -s project` and retry)_. **Do NOT add an "N/A" option** — one option is sufficient for this case.

### 2c — Fetch project board fields (if project selected)

**Run only if user selected a project (not "None / Skip").**

Fetch that project's fields to discover available Type and Priority options:

```bash
gh project field-list {{project-number}} \
  --owner "{{owner}}" \
  --format json
```

Parse the JSON. Look for fields named `Type` and `Priority` (case-insensitive). Extract:
- `field.id` — node ID
- `field.options[]` — array of `{ id, name }` (single-select fields)

Store:
- `projectTypeFieldId` (or `null`)
- `projectTypeOptions` — array of `{ id, name }` with the REAL board options
- `projectPriorityFieldId` (or `null`)
- `projectPriorityOptions` — array of `{ id, name }` with the REAL board options

Also fetch project node ID:

```bash
gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    organization(login: $owner) { projectV2(number: $number) { id } }
  }' -f owner="{{owner}}" -F number={{project-number}} \
  --jq '.data.organization.projectV2.id' 2>/dev/null \
|| gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    user(login: $owner) { projectV2(number: $number) { id } }
  }' -f owner="{{owner}}" -F number={{project-number}} \
  --jq '.data.user.projectV2.id'
```

Store as `projectNodeId`.

**Ask the user for board field values** using `AskUserQuestion` with up to 2 questions. Only ask for fields that actually exist on the project:

**IMPORTANT — Always present the REAL options fetched from the board. Never use hardcoded values here.**

- If `projectTypeFieldId` found: ask **"Type (project board)"** — single-select, **mandatory** (no "None / Skip"). List ALL real option names from `projectTypeOptions` (max 4). User must pick one.
- If `projectPriorityFieldId` found: ask **"Priority (project board)"** — single-select. First option: `None / Skip`. Then list real option names from `projectPriorityOptions` (max 3, totaling 4). Pre-highlight the option that best matches the task type or context.

If neither field exists on the project, skip silently and proceed.

Store selected option IDs as `selectedTypeOptionId` and `selectedPriorityOptionId` (null if skipped).

Show confirmation line:

> "Board fields: Type → **{{name or "—"}}** | Priority → **{{name or "—"}}**"

### 2d — Mandatory description

After receiving all question answers, ask the user in plain text (free-form — NOT AskUserQuestion):

> "Describe the problem or task in detail. What is happening? What should happen instead? Include steps to reproduce if applicable."

**This description is required.** If the user sends nothing meaningful (empty or single word), prompt once more:

> "A detailed description is required. Please describe the problem, expected behavior, and any relevant context."

Use the user's description as the issue body. Do **not** use this reply to change the title.

---

## Step 3 — Resolve GitHub issue type for `--type` flag

Map the **Q3 answer** (GitHub issue type selected by user in Step 2b) to the `--type` value for `gh issue create`:

- Use the exact name selected by the user (e.g. `"Feature"`, `"Bug"`, `"Task"`).
- Store as `ghIssueTypeName`.
- Show: `Issue type: **{{ghIssueTypeName}}**`

If user picked an option not in `repoIssueTypes` (e.g. selected via "Other" free text), use the value as-is and warn that it may fail if not recognized by GitHub.

---

## Step 4 — Create GitHub issue

Build the issue body:

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

Run:

```bash
gh issue create \
  --repo {{owner}}/{{repo}} \
  --title "{{title}}" \
  --body "{{body}}" \
  --assignee "{{gh-username}}" \
  --label "{{label}}" \
  --type "{{ghIssueTypeName}}"
```

> If `gh` returns an error for `--type` (flag not supported or type not found), retry without it and warn the user to set the type manually.

**Label mapping** (create label first if it doesn't exist using `gh label create`):

| Type     | Label name      | Color    |
|----------|-----------------|----------|
| feat     | `enhancement`   | `#a2eeef` |
| fix      | `bug`           | `#d73a4a` |
| refactor | `refactor`      | `#e4e669` |
| chore    | `chore`         | `#cfd3d7` |
| docs     | `documentation` | `#0075ca` |
| test     | `test`          | `#bfd4f2` |
| perf     | `performance`   | `#0e8a16` |
| build    | `build`         | `#e99695` |
| ci       | `ci`            | `#f9d0c4` |
| style    | `style`         | `#fef2c0` |

To get the current GitHub username:
```bash
gh api user --jq '.login'
```

Store the created issue number from the command output (e.g., `https://github.com/owner/repo/issues/42` → `42`).

---

### Step 4a — Upload images (if hasImages is true)

**Run only if `hasImages = true`.**

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

For each file in `pendingImages`, rename to `issue-{{issue-number}}-img-{{N}}.png`, then:

```bash
gh release upload issue-assets "issue-{{issue-number}}-img-{{N}}.png" \
  --repo {{owner}}/{{repo}} \
  --clobber
```

Fetch download URLs:

```bash
gh release view issue-assets \
  --repo {{owner}}/{{repo}} \
  --json assets \
  --jq '.assets[] | select(.name | startswith("issue-{{issue-number}}-")) | .browser_download_url'
```

Store as `imageUrls`.

**4a-3: Update issue body** with real image URLs:

```bash
gh issue edit {{issue-number}} \
  --repo {{owner}}/{{repo}} \
  --body "{{updated-body-with-real-image-urls}}"
```

Log: `✓ Uploaded {{N}} image(s) and attached to issue #{{issue-number}}`

If upload fails, add note in issue body: `> ⚠️ Screenshots could not be uploaded automatically. Please attach them manually.`

---

**If the user selected a project (not "None / Skip"):**

```bash
gh project item-add {{project-number}} \
  --owner "{{owner}}" \
  --url "https://github.com/{{owner}}/{{repo}}/issues/{{issue-number}}"
```

Then fetch the item node ID:

```bash
gh project item-list {{project-number}} \
  --owner "{{owner}}" \
  --format json \
  --jq '.items[] | select(.content.number == {{issue-number}}) | .id'
```

Store as `issueItemId`.

**Set Type and Priority fields** using the REAL option IDs collected in Step 2c:

```bash
# Set Type field (always set — mandatory)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{issueItemId}}" \
  --field-id "{{projectTypeFieldId}}" \
  --single-select-option-id "{{selectedTypeOptionId}}"

# Set Priority field (only if user made a selection)
gh project item-edit \
  --project-id "{{projectNodeId}}" \
  --id "{{issueItemId}}" \
  --field-id "{{projectPriorityFieldId}}" \
  --single-select-option-id "{{selectedPriorityOptionId}}"
```

Log each result: `✓ Set Type → {{name}}` / `✓ Set Priority → {{name}}`.

---

## Step 5 — Final summary

```
✅ Issue created successfully
─────────────────────────────────────────
Issue     #{{issue-number}} — {{title}}
Type      {{semantic-type}} | GitHub: {{ghIssueTypeName}} | Board: {{board-type-name or "—"}}
Priority  {{board-priority-name or "—"}}
Project   {{project-name or "—"}}
─────────────────────────────────────────
Ready to start? Run: /start-task {{issue-number}}
```

---

## Notes

- **This skill works in any GitHub repository.** It auto-detects owner, repo, branches, projects, and available issue types.
- Missing `project` scope degrades gracefully — issue is still created; board fields are skipped with a warning.
- If the repository has no projects, show only "None / Skip" and skip all project-related steps.
- If `gh` is not authenticated, stop at Step 1 and tell the user to run `gh auth login`.
- The PostToolUse hook may auto-assign the issue — expected behaviour; do not re-assign to avoid duplicates.
