---
description: Given a feature issue number, discovers all linked sub-tasks across repositories, creates the feature branch in each affected repo, and opens a draft PR. Run after /create-feature.
---

# Workflow: Start Feature

Given a parent feature issue number, discovers all sub-tasks linked to it (across any repos), creates a consistent `feature/<slug>` branch in each affected repository, adds an initial commit, and opens a draft PR. Works across any GitHub organization.

Run `/create-feature` first to create the parent issue and sub-tasks. Then run `/start-feature <issue-number>`.

---

## Step 1 — Detect repository context and resolve feature issue

Run in parallel:

```bash
gh repo view --json owner,name,defaultBranchRef
gh api user --jq '.login'
```

**Resolve issue number** from `$ARGUMENTS`:
- If `$ARGUMENTS` is a number or `#N`, use it as `featureIssueNumber` in the current repo.
- If `$ARGUMENTS` is `owner/repo#N` or `owner/repo/issues/N`, extract `homeOwner`, `homeRepo`, and `featureIssueNumber`.
- If `$ARGUMENTS` is empty, ask: *"What is the feature issue number?"*

Store: `currentOwner`, `currentRepo`, `defaultBranch`, `ghUsername`.

If `homeOwner`/`homeRepo` not extracted from arguments, default to `currentOwner`/`currentRepo`.

---

## Step 2 — Fetch feature issue and its sub-tasks

Fetch the parent feature issue:

```bash
gh api repos/{{homeOwner}}/{{homeRepo}}/issues/{{featureIssueNumber}} \
  --jq '{ number: .number, title: .title, labels: [.labels[].name] }'
```

Store `featureTitle` and `featureLabels`.

Fetch all sub-issues linked to the parent:

```bash
gh api repos/{{homeOwner}}/{{homeRepo}}/issues/{{featureIssueNumber}}/sub_issues \
  --jq '[.[] | { number: .number, title: .title, repo: .repository.name, owner: .repository.owner.login, issueUrl: .html_url }]'
```

Store as `subTasks` (array of `{ number, title, repo, owner, issueUrl }`).

If the API returns empty or errors:
- Warn: "No sub-tasks found linked to feature #{{featureIssueNumber}}. Sub-issues API may not be available or the issue has no linked sub-tasks."
- Ask: *"Proceed anyway with just the home repo ({{homeRepo}}), or abort?"*
- If proceed: set `subTasks = [{ number: featureIssueNumber, repo: homeRepo, owner: homeOwner }]`

**Build `affectedRepos`**: deduplicate `subTasks` by `owner/repo`. Each entry: `{ owner, repo, issueNumber, issueUrl }`.

Show preview:

```
Feature   #{{featureIssueNumber}} — {{featureTitle}}
Affected repos ({{count}}):
{{for each affectedRepo}}
  • {{owner}}/{{repo}} — linked issue #{{issueNumber}}
{{/for}}
```

---

## Step 3 — Derive branch name

Apply naming convention from the feature title:

1. Lowercase the title
2. Remove special characters (keep only letters, numbers, spaces, hyphens)
3. Replace spaces with `-`
4. Truncate to 50 chars, trim trailing `-`
5. Prefix with `feature/`

Example: `"implement user registration and authentication with AWS Cognito via BFF"` → `feature/implement-user-registration-and-auth`

Store as `branchName`.

Show: `Branch name: **{{branchName}}**`

---

## Step 4 — Confirm scope

Ask via `AskUserQuestion`:

1. **Branch name** (single-select — confirm or override, max 4 options):
   - `{{branchName}}` (derived — Recommended)
   - Up to 3 alternatives with shorter slugs if the derived name is long

   > If the name is ≤35 chars and unambiguous, skip this question and proceed automatically.

---

## Step 5 — Create feature branch in each affected repo

Process repos **sequentially**. For each `{ owner, repo, issueNumber }` in `affectedRepos`:

Log: `Creating branch {{branchName}} in {{owner}}/{{repo}}...`

### 5a — Check if branch already exists

```bash
gh api repos/{{owner}}/{{repo}}/git/ref/heads/{{branchName}} --jq '.ref' 2>/dev/null
```

If exists: log `⚠️  Branch {{branchName}} already exists in {{owner}}/{{repo}} — skipping creation.` and go to Step 5d (just ensure PR exists).

### 5b — Create branch via `gh issue develop`

Use `gh issue develop` to create the branch **and** register the Development section link between the branch and the sub-task issue:

```bash
gh issue develop {{issueNumber}} \
  --repo {{owner}}/{{repo}} \
  --name {{branchName}} \
  --base main
```

> `gh issue develop` without `--checkout` creates the remote branch and links it to the issue in GitHub's Development section — which `git checkout -b` alone cannot do.

If this fails (e.g. branch already exists or issue develop not supported), fall back to API branch creation:

```bash
MAIN_SHA=$(gh api repos/{{owner}}/{{repo}}/git/ref/heads/main --jq '.object.sha')
gh api repos/{{owner}}/{{repo}}/git/refs \
  -f ref="refs/heads/{{branchName}}" \
  -f sha="$MAIN_SHA"
```

### 5c — Add initial commit via GitHub API

GitHub blocks PR creation between identical branches. Create an empty init commit:

```bash
BRANCH_SHA=$(gh api repos/{{owner}}/{{repo}}/git/ref/heads/{{branchName}} --jq '.object.sha')
TREE_SHA=$(gh api repos/{{owner}}/{{repo}}/git/commits/$BRANCH_SHA --jq '.tree.sha')
NEW_SHA=$(gh api repos/{{owner}}/{{repo}}/git/commits \
  -f message="🔧 chore(auth): initialize {{branchName}} branch

AFFECT TASK: #{{issueNumber}}" \
  -f tree="$TREE_SHA" \
  -f "parents[]=$BRANCH_SHA" \
  --jq '.sha')
gh api repos/{{owner}}/{{repo}}/git/refs/heads/{{branchName}} \
  -X PATCH -f sha="$NEW_SHA" -f force=false
```

### 5d — Open draft PR

Determine label from sub-task issue:

```bash
gh api repos/{{owner}}/{{repo}}/issues/{{issueNumber}} --jq '[.labels[].name] | first // "enhancement"'
```

Build PR body:

```markdown
## Summary

{{2-4 bullet points from the sub-task issue body — extract from the "## Context" or "## Scope" section}}

## Related

- Parent feature: https://github.com/{{homeOwner}}/{{homeRepo}}/issues/{{featureIssueNumber}}
- Sub-task: {{issueUrl}}

## ⚠️ Breaking Change

{{extract from sub-task issue body "## ⚠️ Breaking Change" section, or "None." if absent}}

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

Create PR:

```bash
gh pr create \
  --repo {{owner}}/{{repo}} \
  --title "✨ feat({{scope}}): {{featureTitle truncated to 60 chars}}" \
  --base main \
  --head {{branchName}} \
  --draft \
  --assignee "{{ghUsername}}" \
  --label "{{label}}" \
  --body "{{pr-body}}"
```

> **Scope**: derive from repo name — strip `fluxup-` prefix. E.g. `fluxup-backend` → `backend`, `fluxup-bff` → `bff`.

Store result: `{ owner, repo, issueNumber, branchName, prUrl, prNumber }`.

### 5e — Verify Development section link

```bash
gh issue develop {{issueNumber}} \
  --repo {{owner}}/{{repo}} \
  --list 2>/dev/null | grep "{{branchName}}"
```

If branch appears in output: log `✓ {{owner}}/{{repo}}#{{issueNumber}} linked to branch {{branchName}}`

If not: log `⚠️  Development link not detected for {{owner}}/{{repo}}#{{issueNumber}}. Open the PR sidebar → Development → Link an issue → #{{issueNumber}}`

---

## Step 6 — Save feature metadata

After all repos are processed, persist metadata for use by other skills (e.g. `ship-task`, `ship-feature`).

**File path:** `~/.claude/tasks/{{homeRepo}}/{{branchName}}.json`
(Windows: `C:\Users\<user>\.claude\tasks\{{homeRepo}}\{{branchName}}.json`)

Create parent directories if needed. Write with the Write tool — do NOT use shell redirection.

```json
{
  "username": "{{ghUsername}}",
  "featureIssueNumber": {{featureIssueNumber}},
  "featureIssueUrl": "https://github.com/{{homeOwner}}/{{homeRepo}}/issues/{{featureIssueNumber}}",
  "featureTitle": "{{featureTitle}}",
  "branchName": "{{branchName}}",
  "homeOwner": "{{homeOwner}}",
  "homeRepo": "{{homeRepo}}",
  "createdAt": "{{ISO-8601 timestamp}}",
  "repos": [
    {
      "owner": "{{owner}}",
      "repo": "{{repo}}",
      "issueNumber": {{issueNumber}},
      "issueUrl": "{{issueUrl}}",
      "prUrl": "{{prUrl}}",
      "prNumber": {{prNumber}}
    }
  ]
}
```

---

## Step 7 — Final summary

```
✅ Feature branches created successfully
─────────────────────────────────────────
Feature   {{homeOwner}}/{{homeRepo}}#{{featureIssueNumber}} — {{featureTitle}}
Branch    {{branchName}}

Repos:
{{for each repo result}}
  • {{owner}}/{{repo}}#{{issueNumber}} — PR #{{prNumber}} [DRAFT]
    {{prUrl}}
{{/for}}
─────────────────────────────────────────
Next: run /start-task <sub-issue-number> on each repo to begin implementation.
      Each sub-task branch should target {{branchName}}, not main.
```

---

## Notes

- **Branch name is identical across all repos** — consistency is required for cross-repo coordination.
- Sub-tasks discovered via GitHub's sub-issues API (`repos/{owner}/{repo}/issues/{N}/sub_issues`). If this API is unavailable (org plan limitation), warn and fall back to manual repo list.
- `gh issue develop` establishes the Development section link between branch and issue. This is the only programmatic way to do so for non-default-branch targets.
- Initial commits are created via GitHub API (no local clone needed) — the branch content is identical to `main` at this point.
- PRs are created as **draft** — they become ready for review only after implementation is complete (use `/ship-task` per sub-task).
- If a PR already exists for `{{branchName}}` in a repo, skip creation and log the existing PR URL.
- The PostToolUse hook may auto-assign PRs — expected behaviour; do not re-assign.
- If `gh` is not authenticated, stop at Step 1 and tell the user to run `gh auth login`.
