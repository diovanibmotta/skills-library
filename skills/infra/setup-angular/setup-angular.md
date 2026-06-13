# Angular Project Setup

Prepare the environment to run an Angular frontend project. This skill reads project configuration files, detects required versions, installs missing tools, and installs dependencies.

## Steps

Execute the following steps **in order**. Stop and report clearly if any step fails.

---

### Step 1 — Read project requirements

Read `package.json` in the current working directory.

Extract:
- `engines.node` if present (explicit Node.js version constraint)
- `@angular/core` version from `dependencies`
- `@angular/cli` version from `devDependencies`
- `typescript` version from `devDependencies`

Also check for `.nvmrc` or `.node-version` files in the project root. If found, their content overrides the Node.js version target.

Use this table to derive the **minimum required Node.js LTS version** when `engines.node` is absent:

| Angular version | Minimum Node.js |
|---|---|
| 19.x | 18.19.1 LTS or 20.x LTS |
| 18.x | 18.13.0 LTS |
| 17.x | 18.13.0 LTS |
| 16.x | 16.14.0 LTS |

---

### Step 2 — Check Node.js

Run:
```bash
node --version 2>/dev/null || echo "NOT_INSTALLED"
```

Compare the installed version against the requirement from Step 1.

**If Node.js is not installed or version is too old:**

Check if `nvm` is available:
```bash
command -v nvm || command -v nvm.exe || [ -s "$HOME/.nvm/nvm.sh" ] && echo "NVM_AVAILABLE" || echo "NVM_NOT_FOUND"
```

- **If nvm is available:** Install and activate the correct version:
  ```bash
  nvm install <required-version>
  nvm use <required-version>
  ```

- **If nvm is NOT available on Windows:** Use `winget` to install Node.js:
  ```bash
  winget install OpenJS.NodeJS.LTS
  ```
  Then instruct the user to close and reopen the terminal, and run `/setup-angular` again.

- **If nvm is NOT available on Linux/macOS:** Use the official install script or instruct the user to install via their package manager.

**If Node.js version is already correct:** report it as OK and continue.

---

### Step 3 — Check npm

Run:
```bash
npm --version 2>/dev/null || echo "NOT_INSTALLED"
```

npm ships with Node.js — if Node is installed, npm should be present. If it is missing, report it and stop.

---

### Step 4 — Check Angular CLI

Run:
```bash
ng version 2>/dev/null | head -3 || echo "NOT_INSTALLED"
```

Read the required `@angular/cli` version from `package.json` `devDependencies` (e.g., `^19.2.22` → major `19`).

**If Angular CLI is not installed globally:** install it:
```bash
npm install -g @angular/cli@<major-version>
```

**If Angular CLI is installed but the major version differs from the project:** update it:
```bash
npm install -g @angular/cli@<major-version>
```

**If Angular CLI major version matches:** report as OK.

---

### Step 5 — Install project dependencies

Check if `node_modules` already exists:
```bash
[ -d node_modules ] && echo "EXISTS" || echo "MISSING"
```

**If `node_modules` is missing or the user passed `--fresh`:** run:
```bash
npm install
```

**If `node_modules` exists:** run:
```bash
npm install --prefer-offline
```

Report the exit code. If `npm install` fails, show the last 20 lines of output and stop.

---

### Step 6 — Verify the setup

Run a quick build check (dry-run) to confirm everything is wired up:
```bash
npx ng build --configuration development 2>&1 | tail -20
```

If the build succeeds (exit 0), the environment is ready.
If it fails, show the full error output and suggest fixes based on the error message.

---

### Step 7 — Final report

Print a summary table:

```
Environment ready for: <project-name from package.json>
------------------------------------------------------------
Node.js        <version>    ✓ / needs attention
npm            <version>    ✓ / needs attention
Angular CLI    <version>    ✓ / needs attention
Dependencies   installed    ✓ / needs attention
Build check    passed       ✓ / needs attention
------------------------------------------------------------
Run with:  npm start   (ng serve)
Test with: npm test
Build:     npm run build
```

If anything needs attention, list the exact commands the user must run manually (e.g., opening a new terminal after a PATH change).
