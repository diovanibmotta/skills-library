# setup-angular

Prepares the environment to run an Angular frontend project. Reads `package.json`, detects required versions, installs missing tools, and installs dependencies.

## Trigger

```
/setup-angular
```

Optional flag:

```
/setup-angular --fresh
```

Forces a clean `npm install` even if `node_modules` already exists.

## What it does

1. Reads `package.json` to extract Angular version, Node.js constraint, and TypeScript version
2. Checks the installed Node.js version against the requirement; installs the correct version via `nvm` or `winget` if needed
3. Verifies `npm` is available
4. Checks the Angular CLI global version; installs or upgrades to match the project's major version
5. Installs project dependencies (`npm install` or `npm install --prefer-offline`)
6. Runs a development build to confirm the setup is correct
7. Prints a summary table with version status and run commands

## Prerequisites

- Node.js package manager (`nvm` recommended, or `winget` on Windows)
- `npm`

## Angular ↔ Node.js version matrix

| Angular | Minimum Node.js |
|---------|----------------|
| 19.x | 18.19.1 LTS or 20.x |
| 18.x | 18.13.0 LTS |
| 17.x | 18.13.0 LTS |
| 16.x | 16.14.0 LTS |

## Usage

```
/setup-angular
```

Run in the Angular project root (where `package.json` lives).
