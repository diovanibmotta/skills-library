---
description: Create or edit GitHub wiki pages. Lists repos with wiki enabled, lets user select repo and page, then writes content via git clone of the wiki repo.
---

# Workflow: GitHub Wiki Manager

Create new wiki pages or edit existing ones across any GitHub repository.

---

## Step 1 — Detect current context and list repos with wiki

Run in parallel:

```bash
# Get current repo context (may not exist if outside a git repo)
gh repo view --json owner,name 2>/dev/null || echo "NOT_IN_REPO"

# List all repos the user owns or has access to, with wiki status
gh api user/repos --paginate --jq '.[] | {name: .name, owner: .owner.login, has_wiki: .has_wiki, full_name: .full_name}' 2>/dev/null
```

Also list org repos if applicable:
```bash
gh api user/orgs --jq '.[].login' 2>/dev/null
```

For each org found (up to 3), fetch:
```bash
gh api orgs/{org}/repos --jq '.[] | {name: .name, owner: .owner.login, has_wiki: .has_wiki, full_name: .full_name}' 2>/dev/null
```

Collect all repos. Separate into two lists:
- `reposWithWiki` — repos where `has_wiki == true`
- `reposWithoutWiki` — all others (wiki may be disabled in settings)

If the current directory is inside a git repo, note that repo and put it first in the list.

Store the full deduplicated list. If more than 8 repos total, limit to 8 most recently updated (they come out of the API sorted by `pushed_at`).

---

## Step 2 — Ask user to select repository

Use `AskUserQuestion` to ask:

1. **Repository** (single-select, max 4 options):
   - List up to 3 repos from `reposWithWiki`. Label each as `{owner}/{repo}`.
   - If current repo is in `reposWithWiki`, put it first and append "(current)".
   - 4th option: if there are more repos not listed, show "Other (type name)" — handle via user free text.
   - If no repos have wiki enabled: show all options with a note that wiki must be enabled in repo settings first.

Show this note before the question:
> "Repos with wiki disabled can still be targeted — GitHub will create the wiki on first push."

---

## Step 3 — Clone wiki repo to temp dir

Get the GitHub auth token and clone:

```bash
# Get token
TOKEN=$(gh auth token)

# Set temp dir
WIKI_DIR="$env:TEMP\wiki-{repo}-$(Get-Random)"

# Clone wiki repo (wiki is a separate git repo at repo.wiki.git)
git clone "https://oauth2:${TOKEN}@github.com/{owner}/{repo}.wiki.git" "$WIKI_DIR" 2>&1
```

On Windows (PowerShell):
```powershell
$TOKEN = gh auth token
$WIKI_DIR = "$env:TEMP\wiki-{repo}-$(Get-Random)"
git clone "https://oauth2:$TOKEN@github.com/{owner}/{repo}.wiki.git" $WIKI_DIR 2>&1
```

On Linux/Mac (Bash):
```bash
TOKEN=$(gh auth token)
WIKI_DIR="/tmp/wiki-{repo}-$$"
git clone "https://oauth2:${TOKEN}@github.com/{owner}/{repo}.wiki.git" "$WIKI_DIR" 2>&1
```

**If clone fails with "repository not found" or "empty repository":** the wiki has never been initialized. Set `wikiIsNew = true`. Create the directory and initialize a new git repo:

```bash
# Windows
New-Item -ItemType Directory -Path $WIKI_DIR -Force
Set-Location $WIKI_DIR
git init
git remote add origin "https://oauth2:$TOKEN@github.com/{owner}/{repo}.wiki.git"
```

Store `WIKI_DIR` and `wikiIsNew`.

---

## Step 4 — List existing wiki pages

```bash
# List all .md files in the wiki dir (each file = one wiki page)
ls "$WIKI_DIR"/*.md 2>/dev/null || echo "NO_PAGES"
```

Parse filenames. GitHub wiki page titles come from filenames:
- `Home.md` → "Home" (root/default page)
- `Getting-Started.md` → "Getting Started"
- `API-Reference.md` → "API Reference"

Convert filenames to display titles: replace `-` with space, remove `.md` extension.

Store `existingPages` as list of `{ filename, title }`.

---

## Step 5 — Ask user: create new or edit existing

Use `AskUserQuestion` with 1 question:

**If `existingPages` is empty or `wikiIsNew`:**

1. **Action** (single-select):
   - `Create new page` — write a new wiki page
   - `Create Home page` — create the default Home.md entry point

**If `existingPages` has pages:**

1. **Page to work on** (single-select, max 4 options):
   - List up to 3 existing pages by title (put "Home" first if present)
   - Last option: `+ Create new page`
   - If more than 3 pages exist: put the 3 most relevant and note "Other pages exist — type the page name in the next step"

Store selection as `selectedAction` = "edit" or "create", and `selectedPage` = filename or null.

---

## Step 6 — Get current content (edit only)

**If editing an existing page:**

Read the current content of the selected page file:

```bash
# Read current wiki page content
cat "$WIKI_DIR/{selectedPage}"
```

Show the content to the user:

> **Current content of "{{title}}":**
> ```markdown
> {{content}}
> ```

Then ask in plain text (free-form — NOT AskUserQuestion):

> "What changes should be made to this page? Paste the new full content, or describe the edits and I'll apply them."

Wait for the user's response. If the user pastes full markdown content, use it directly. If they describe edits (e.g. "add a section about X", "fix the typo in paragraph 2"), apply the edits to the existing content.

---

## Step 7 — Get new page info (create only)

**If creating a new page:**

Ask in plain text (free-form):

> "What is the title for the new page?"

Wait for title. Then ask:

> "Paste the markdown content for '{{title}}':"

Wait for content.

Convert the title to a filename: replace spaces with `-`, preserve case, add `.md`.
- "Getting Started" → `Getting-Started.md`
- "API Reference" → `API-Reference.md`
- "Home" → `Home.md`

Store as `newFilename` and `newContent`.

---

## Step 8 — Write and commit the page

**For edits:**

```bash
# Overwrite the file
Set-Content -Path "$WIKI_DIR/{selectedPage}" -Value "{newContent}" -Encoding utf8
```

**For new pages:**

```bash
Set-Content -Path "$WIKI_DIR/{newFilename}" -Value "{newContent}" -Encoding utf8
```

**On Linux/Mac use:**
```bash
cat > "$WIKI_DIR/{filename}" << 'WIKI_EOF'
{content}
WIKI_EOF
```

Then configure git identity (required for wiki repos) and commit:

```bash
cd "$WIKI_DIR"
git config user.email "$(gh api user --jq '.email // "noreply@github.com"')"
git config user.name "$(gh api user --jq '.name // .login')"
git add "{filename}"
git commit -m "{action}: {title}"
```

Where `{action}` is `docs` for edits and `feat` for new pages.

---

## Step 9 — Push

```bash
cd "$WIKI_DIR"
git push origin HEAD:master 2>&1 || git push origin HEAD:main 2>&1
```

GitHub wiki repos use `master` as default branch. If that fails, try `main`.

**If push fails due to diverged history:**

```bash
git pull --rebase origin master 2>&1 && git push origin HEAD:master 2>&1
```

---

## Step 10 — Cleanup and final summary

Remove the temp clone:

```bash
# Windows
Remove-Item -Recurse -Force "$WIKI_DIR" 2>/dev/null

# Linux/Mac
rm -rf "$WIKI_DIR"
```

Print summary:

```
✅ Wiki page {{action}}d successfully
─────────────────────────────────────
Repo    {owner}/{repo}
Page    {title}
Action  {Created / Updated}
URL     https://github.com/{owner}/{repo}/wiki/{page-slug}
─────────────────────────────────────
```

Where `{page-slug}` is the filename without `.md`, with `-` preserved (GitHub uses this in URLs).

---

## Notes

- GitHub wiki is a **separate git repo** at `https://github.com/{owner}/{repo}.wiki.git` — changes via git, not the REST API.
- Wiki must be enabled in repo Settings → Features → Wikis. If never initialized, the first push creates it.
- The `Home.md` file is always the wiki's root/landing page.
- Page filenames use hyphens as word separators; GitHub auto-converts to spaces in the UI.
- If `gh auth token` fails, instruct user to run `gh auth login` first.
- Never store the auth token beyond the current command — pass it inline to git clone URL only.
- If the user wants to add images to a wiki page, they must be uploaded separately (GitHub wiki doesn't support direct image upload via API) — note this and suggest linking to images hosted in the repo's Issues or Releases.
