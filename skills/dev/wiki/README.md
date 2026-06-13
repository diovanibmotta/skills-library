# wiki

Create new wiki pages or edit existing ones across any GitHub repository via git clone of the wiki repo.

## Trigger

```
/wiki
```

## What it does

1. Lists all repositories the user owns or has access to, filtered by wiki enabled
2. Asks the user to select a repository
3. Clones the wiki git repo (`repo.wiki.git`) to a temp directory
4. Lists existing wiki pages and lets the user choose to create or edit one
5. Accepts the new/updated markdown content from the user
6. Commits and pushes back to the wiki repo
7. Cleans up the temp clone

## Prerequisites

- `gh` authenticated (`gh auth login`)
- `git` installed
- Wiki enabled in the target repo's Settings → Features → Wikis

## Usage

```
/wiki
```

Follow the interactive prompts to select the repo, page, and content.

## Notes

- The GitHub wiki is a separate git repo — this skill manages it via git, not the REST API
- `Home.md` is always the wiki's root/landing page
- If the wiki has never been initialized, the first push creates it automatically
- Images must be hosted externally (Issues, Releases) and linked — direct wiki image upload is not supported via API
