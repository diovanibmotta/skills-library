---
description: Runs a structured security review over the code changes of a branch/PR/diff, language- and stack-agnostic (application code, infrastructure as code, containers, CI/CD). Applies OWASP Top 10 controls, checks secrets management, input validation, injection prevention (SQL/NoSQL/command/XSS), authentication/authorization (IDOR, JWT, password hashing), sensitive data exposure, error handling/logging, rate limiting, dependency supply chain, and infrastructure configuration (Dockerfile, Kubernetes, Terraform, IAM, pipelines). Classifies each finding on a severity scale (🔴 CRITICAL to ⚪ INFO), delivers a ready-to-apply fix, and produces a versioned .md report. Use whenever the user asks to "review code security", "validate the security of these changes", "run a security analysis on this diff/branch/PR", "check OWASP", "audit vulnerabilities", "run a security review before merge", or any variation of validating security best practices over already-written code before opening/merging a PR. Complements (does not replace) automated scanners (SAST/SCA/tfsec/checkov) — the differentiator of this skill is the structured report with severity, CWE/OWASP mapping, contextualized impact, and a ready-to-apply fix, plus explicit coverage of infrastructure as code.
allowed-tools: Read, Write, Bash, Grep, Glob, AskUserQuestion
---

# Security Review of Code Changes

Language-, stack-, and cloud-provider-agnostic workflow — applies to any repository (Java, Python, Angular/TypeScript, Go, Terraform, Docker, Kubernetes, CI/CD pipelines). Treat every reviewed change as **real production code**: exposed to the internet, receiving potentially malicious input, and possibly carrying data subject to privacy/compliance regulation.

## Role

Act as a senior security analyst reviewing the change on behalf of the security team. For each finding: point to the exact file and line, explain the impact **in this application's real context** (not generic), deliver a ready-to-apply fix, and classify severity. When a piece of code is adequate, confirm this explicitly — don't silently omit reviewed-and-approved parts.

## Precedence: security conventions already documented in the repository

Before applying the generic checklists in this skill, check whether the repository (or the broader workspace) already documents its own security rules — `CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`, `SECURITY.md`, or an equivalent security-guidance file. Repository-specific rules take precedence over this skill's generic recommendations when they conflict (e.g. a repo-specific requirement for deterministic test data is not a generic security rule, but should still be respected as the repo's own floor). Treat this skill as the default for what **isn't** already covered by an established convention.

## Workflow

### Step 1 — Determine the diff scope

Don't assume the comparison baseline without a clear signal. If context already makes it obvious (e.g. the user referenced a specific PR, or there are only uncommitted changes), use that directly; otherwise, ask which ref the diff should be computed against (the repository's default branch, a specific commit, or the branch behind an open PR/MR).

Once the base is confirmed:

1. `git diff --stat <base>...HEAD` (plus uncommitted changes, if any) to list touched files.
2. Ignore build/dependency/vendor directories (`node_modules`, `.git`, `vendor`, `dist`, `build`, `.terraform`, `target`, lock-file caches irrelevant to the functional diff) — but **don't** ignore dependency manifests (`package.json`, `requirements.txt`, `pom.xml`, `go.mod`, `*.lock`) nor infrastructure files (`Dockerfile`, Kubernetes `*.yaml`, `*.tf`, `.gitlab-ci.yml`/`.github/workflows/*`) — those get their own checklist (Step 5).
3. For each relevant file, get the full diff (not just `--stat`) — the analysis needs the real content, and often the whole file when the change's security depends on surrounding context (e.g. whether a new route has authentication middleware applied a few levels up).

### Step 2 — Detect the stack and select applicable checklists

Classify changed files by type without assuming the whole repo's stack — a single diff can touch application and infrastructure at once:

| Signal in the diff | Load |
|---|---|
| Application code (`.java`, `.py`, `.ts`/`.js`, `.go`, etc.) | "Application code checklist" below |
| Dependency manifest (`package.json`, `requirements.txt`/`pyproject.toml`, `pom.xml`, `go.mod`, lockfiles) | "Dependencies and infrastructure checklist" (Dependencies section) |
| `Dockerfile`, `docker-compose*.yml`, Kubernetes manifests, `.gitlab-ci.yml`/`.github/workflows/*`, `*.tf` | "Dependencies and infrastructure checklist" (Infra/CI/Cloud section) |
| Any file | "OWASP Top 10 and red flags" (absolute rules always apply) |

### Step 3 — Quick pattern scan

Before deep manual reading, run a scan for known high-risk patterns using the "Quick scan patterns" reference below, scoped to the diff's files. This doesn't replace the manual analysis in Step 4 — it's a first filter so obvious findings aren't missed and reading time is used efficiently.

### Step 4 — Manual, contextual review

For each application file in scope, using the "Application code checklist" categories (secrets/config, input validation, injection, authentication/authorization, data exposure/cryptography, error handling/logging, rate limiting):

- Don't limit yourself to changed lines when the change's security depends on context outside the diff — e.g. to confirm a new route is actually covered by authentication middleware, or that a resource ID in the URL is validated against the authenticated user (IDOR), it may require `Grep`/`Read` on unchanged files.
- Trace user input data flow from entry point to its use (query, system command, HTML output) whenever the diff introduces or modifies that flow.
- For APIs: check whether the response leaks unnecessary fields (`password_hash`, `internal_id`) and whether errors returned to the client are generic (stack trace only in server logs).
- If you find an existing `# TODO:SECURITY` comment (or equivalent) already flagging an accepted risk, treat it as a **documented** risk — verify the described mitigation is actually sufficient, but don't treat it as an undiscovered finding; flag it if the comment exists but the mitigation is missing/incomplete.

### Step 5 — Dependencies and infrastructure

When the diff touches dependency manifests or infrastructure files, apply the checklist below:

- **Dependencies**: pinned versions vs. open ranges/`latest`, typosquatting, suspicious `postinstall`/`preinstall` scripts, known CVEs in versions in use.
- **Infra/containers/K8s/CI/Cloud**: non-root container user, `securityContext`, plaintext secrets in manifests/pipelines, IAM policies with wildcard `Action`/`Resource`, security groups open on sensitive ports, third-party actions/steps not pinned by immutable hash/tag.

Whenever the corresponding tool is available in the current environment (`npm audit`, `pip-audit`, `mvn dependency-check`, `tfsec`, `checkov`, `tflint`, secret scanners), **actually run it** against the changed files and fold the real result into the finding, instead of only inferring by reading — treat this as a floor, not an optional extra, whenever the target repository already requires it as a CI gate.

### Step 6 — Classify severity

| Level | Criterion | Expected action |
|---|---|---|
| 🔴 CRITICAL | Remote exploitation, RCE, authentication bypass, massive data leak | Block — cannot ship to production as-is |
| 🟠 HIGH | Requires authentication or a specific condition; significant data impact | Fix in the current sprint |
| 🟡 MEDIUM | Limited impact or complex exploitation; contributes to chained attacks | Fix within 30 days |
| 🔵 LOW | Defense in depth; not independently exploitable | Fix in a planned refactor |
| ⚪ INFO | Quality/security posture, no direct exploitation | Log and evaluate |

Map each finding to a CWE and to the corresponding OWASP Top 10 category (see table below). Never pad the list with artificial findings just to fill sections — if a category is genuinely adequate, state that explicitly.

#### OWASP Top 10 (2021) mapping → minimum expected control

| # | Category | Minimum expected control in this review |
|---|---|---|
| A01 | Broken Access Control | Authorization on every protected route; horizontal (IDOR) and vertical (privilege) access checked |
| A02 | Cryptographic Failures | TLS mandatory; `bcrypt`/`argon2id` for passwords; sensitive data encrypted at rest; no weak algorithm (MD5/SHA1/DES/RC4/ECB) |
| A03 | Injection | Parameterized queries; HTML output sanitized/escaped; no `eval`/`exec` with user input |
| A04 | Insecure Design | Sensitive functionality (auth, payment, PII) has at least minimal threat modeling before coding — if absent, flag as a process gap, not just a code gap |
| A05 | Security Misconfiguration | Debug disabled in production; security headers configured; CORS restricted to an allow-list |
| A06 | Vulnerable and Outdated Components | Dependencies pinned and free of known unaddressed critical CVEs |
| A07 | Identification and Authentication Failures | Rate limiting on login; lockout/backoff after failed attempts; MFA for admin accounts when the product already supports it |
| A08 | Software and Data Integrity Failures | Checksums/pinning for packages and images; CI pipeline validates artifact origin before publishing |
| A09 | Security Logging and Monitoring Failures | Structured logs for authentication/authorization/errors; no sensitive data in logs; defined retention |
| A10 | Server-Side Request Forgery (SSRF) | Domain allow-list on server-initiated HTTP requests derived from user input; no redirect to a user-controlled URL without validation |

### Step 7 — Generate the report

1. Use the report template below, filling in one finding block per real vulnerability (file:line, category, CWE/OWASP, problem, contextualized impact, fix in code, reference), plus the final summary table.
2. Save to `.ai/security-reviews/<diff-or-branch-slug>/review-<YYYYMMDD-HHMMSSmmm>.md` at the root of the reviewed repository (use the system's real date/time; 3-digit milliseconds avoid collisions). Create the directories if they don't exist.
3. In the front matter, record: diff base (`<base>...<head>` resolved to a commit hash), reviewed files, finding count by severity, and whether any item requires escalation to the security team.

### Step 8 — Present to the operator

In chat, a direct summary — don't repeat the whole report, it's already in the file:

1. Findings table (severity, category, file, line).
2. Top 3 priorities — the highest-risk items to fix first.
3. What couldn't be analyzed (dependency on external context, missing file, tool unavailable in the environment).
4. Escalation recommendation when applicable (see criteria below).
5. Path to the generated report file.

## Application code checklist

### 1. Secrets and configuration management

- Every credential, key, or token comes from an **environment variable** or a vault (Vault, AWS/Azure/GCP secrets manager). Never literal in code, tests, fixtures, or example comments.
- The application should **fail at startup** if a required environment variable is missing — never silently fall back to an insecure default (e.g. `SECRET_KEY = os.getenv("SECRET_KEY", "dev")` is a finding).
- `.env` files or configs with real secrets must not be committed — check `.gitignore` and whether the diff adds one of these files.
- CI/CD secrets must reference the platform's vault (GitLab CI/CD Protected+Masked variables, GitHub Secrets) — never hardcoded in the pipeline file.

### 2. Input validation and sanitization

- All user input is untrusted — validation happens **server-side**, never only client-side.
- Type, format, and min/max length for every relevant field.
- Numeric parameters with explicit min/max bounds (avoids overflow, unexpected negative values, DoS via absurd values).
- Prefer the stack's established validation libraries (`pydantic`/`marshmallow` in Python, `joi`/`zod`/`class-validator` in Node/TS, Bean Validation in Java) over scattered manual validation.
- File upload: real type validated by **magic bytes**, not just the extension/client-declared `Content-Type`; original filename never used directly on the filesystem (replace with a UUID); max size enforced before reading the whole file into memory.

### 3. Injection prevention

- **SQL/NoSQL**: always parameterized queries/prepared statements or the ORM's binding mechanism. Any f-string/concatenation/`.format()` building a query with a value from user input is a 🔴 finding, regardless of language.
- ORM methods that accept raw queries are only acceptable with explicit bind parameters, never string concatenation of input.
- **System command**: `subprocess`/`exec`/`child_process.exec`/`Runtime.exec` must never receive a string built with user input without an explicit argument list (`shell=False` + arg list instead of `shell=True` + string).
- **XSS**: all HTML-rendered output must be escaped by the framework's standard mechanism (template engine auto-escape, Angular's `DomSanitizer` instead of raw `[innerHTML]` with untrusted data, React's `dangerouslySetInnerHTML` always with prior sanitization).
- **Insecure deserialization**: `pickle.loads`, `yaml.load` (without `SafeLoader`), PHP `unserialize`, or Java deserialization from an untrusted source are 🔴 findings — use safe formats/parsers (`json`, `yaml.safe_load`, class-allowlisted libraries).

### 4. Authentication and authorization

- Every route/endpoint that should be protected has the authentication middleware/decorator applied — actively look for "forgotten" routes (compare against sibling routes in the same file/module).
- Authorization checks **ownership of the specific resource**, not just "is authenticated" — resource IDs from the URL/payload must be validated as belonging to the authenticated user (IDOR prevention). This often requires looking beyond the diff: where is the resource loaded, and is the `owner_id`/`tenant_id` filter present?
- **JWT**: signature, expiration (`exp`), issuer (`iss`), and audience (`aud`) must be validated — decoding the payload without verifying the signature is always 🔴.
- Passwords: `bcrypt` (cost factor ≥ 12) or `argon2id`. MD5/SHA1/plaintext for passwords is always 🔴.
- Password reset: identical response for existing and non-existing email (avoids user enumeration).
- Sessions expire on inactivity and are invalidated on logout (token/session revoked server-side, not just a cleared client cookie).
- Mass assignment: fields the user shouldn't be able to set (`role`, `is_admin`, `owner_id`, `price`) can't come from a generic payload bind without an explicit field allow-list.

### 5. Sensitive data exposure and cryptography

- API responses don't return unnecessary fields (`password_hash`, `internal_id`, internal audit fields) — check serializers/DTOs, not the raw model.
- TLS mandatory, no HTTP fallback in production; skipping TLS certificate verification on an HTTPS call is always 🔴.
- Security headers on internet-facing HTTP responses: `Strict-Transport-Security`, `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`.
- CORS with an explicit origin allow-list on authenticated APIs — a wildcard (`*`) is always 🔴 when authentication is involved.
- Sensitive data at rest (database, files, backups) encrypted when it contains PII/financial data.
- Weak algorithms (MD5, SHA1, DES, RC4, AES-ECB) for any cryptographic use (not just passwords) are a finding.
- Non-cryptographic random generators (`random`, `Math.random()`) used for a security token/key/nonce are a finding — use `secrets`/`crypto.randomBytes`/`SecureRandom`.

### 6. Error handling and logging

- Errors returned to the client are generic, without stack trace, SQL query, file path, or exception detail — those details stay in server logs only.
- An empty catch block that silently swallows an error without logging is always a finding — at least 🟡, higher if the hidden error has security implications (e.g. an ignored signature-validation failure).
- Structured logging (ideally JSON) with `timestamp`, `level`, `event`, `user_id`, `ip`, `request_id` when the project already uses structured logging — don't introduce a log format that diverges from the rest of the module.
- **Must never appear in logs**: passwords (even incorrect ones, on login attempts), full authentication tokens (log at most a prefix/hash), financial data (card, bank account), sensitive PII.
- **Should be logged**: authentication attempts (success and failure) with source IP, authorization errors (403) — especially repeated/suspicious patterns, creation/modification/deletion of critical records, user permission/role changes.

### 7. Rate limiting and abuse protection

Reference values (adjust to the endpoint's real context, but question the total absence of a limit on sensitive endpoints):

| Endpoint | Reference limit |
|---|---|
| Login/authentication | 5 attempts/minute per IP |
| Public API | 100 req/minute per IP or API key |
| Password reset | 3 req/hour per email |
| File upload | Max size defined + MIME type/magic-bytes validated |
| Costly operation (export, report) | 1 at a time per user |
| Account creation | CAPTCHA or mandatory email verification |

If the diff introduces a new sensitive endpoint (auth, password reset, costly operation) with no rate limiting and the project's framework/gateway already offers the mechanism, this is a finding — usually 🟡, escalating to 🟠 if the endpoint is public and critical (login, password reset).

## OWASP Top 10 and red flags

### Absolute rules — always flag as 🔴 CRITICAL

```
Hardcoded credential, token, or API key (including in a comment or test fixture)
User input concatenated/f-string'd into a SQL/NoSQL query
eval()/exec() (or equivalent: Function(), vm.runInContext, subprocess with shell=True+string) with user-controllable input
JWT decoded without validating signature, expiration, issuer, and audience
Password hash using MD5, SHA1, or plaintext storage
Disabled TLS certificate verification on an HTTPS request
Stack trace or exception detail returned directly to the client
CORS wildcard (*) on an authenticated API
Container running as root with access to the Docker socket
Secret in a Dockerfile ARG/ENV
Insecure deserialization of untrusted data (pickle, yaml.load without SafeLoader, PHP unserialize)
Empty catch block hiding a validation/authentication failure
```

### Red flags — question before accepting (not necessarily critical, but always review)

```
# For production, add authentication here   (comment standing in for a real implementation)
SECRET = "literal_value" / PASSWORD = "literal_value" / API_KEY = "literal_value"
"This is a simplified version" / "for demo purposes"   (justification for skipping a security control)
Unrestricted CORS / allow_all_origins=True / origins="*"
Decoding a token without validating the JWT signature
Full exception trace included in the client response
DEBUG=True / debug=True in any committed non-local environment config
Non-cryptographic random used to generate a token, key, or security nonce
```

### The `# TODO:SECURITY` convention

If a practice from the list above is technically necessary in a specific context (e.g. an internal endpoint that only runs on an isolated network and can't get rate limiting due to an infrastructure limitation), the code should flag this explicitly with a `# TODO:SECURITY` comment (or the language's equivalent) explaining the risk and the mitigation adopted. When reviewing:

- If the comment exists and the described mitigation is real and sufficient → not a new finding, but confirm the mitigation is actually implemented (not just the comment).
- If the comment exists but the mitigation is insufficient or missing → a finding, severity per the real risk, but note that it was already flagged (relevant for the team to decide whether this is an accepted exception or unresolved debt).
- If the risky practice exists with **no** flag at all → a normal finding, no special treatment.

## Quick scan patterns

Use these with `Grep`/`ripgrep`, scoped to the diff's files, as a first filter before manual reading (Step 3). A match here isn't automatically a confirmed finding — it's a signal to investigate the real context of the line before reporting. Adapt the list to the diff's actual language; don't run every pattern against every file.

**Generic (any language)**
```
(api[_-]?key|secret|password|token|passwd|pwd)\s*[:=]\s*["'][^"'\s]{6,}["']
BEGIN (RSA |EC )?PRIVATE KEY
AKIA[0-9A-Z]{16}                          # AWS access key id
-----BEGIN OPENSSH PRIVATE KEY-----
Authorization:\s*Bearer\s+[A-Za-z0-9\-_.]{20,}
TODO:SECURITY
```

**Python**
```
eval\(|exec\(
pickle\.loads?\(
yaml\.load\((?!.*SafeLoader)
subprocess\.(run|call|Popen).*shell\s*=\s*True
os\.system\(
verify\s*=\s*False
except\s+Exception\s*:\s*pass
except\s*:\s*pass
random\.(random|randint)\(.*(token|secret|password|key)
hashlib\.(md5|sha1)\(
DEBUG\s*=\s*True
```

**JavaScript / TypeScript (Node, Angular)**
```
eval\(
new Function\(
child_process\.(exec|execSync)\(
dangerouslySetInnerHTML
\[innerHTML\]\s*=
document\.write\(
rejectUnauthorized\s*:\s*false
cors\(\)\s*\)          # no origin options
origin\s*:\s*["']\*["']
Math\.random\(\).*(token|secret|password|key)
console\.(log|error)\(.*(password|token|secret)
```

**Java**
```
Runtime\.getRuntime\(\)\.exec\(
ProcessBuilder\(.*\+
Class\.forName\(
ObjectInputStream
MessageDigest\.getInstance\("MD5"\)
MessageDigest\.getInstance\("SHA-1"\)
\.setVerifyHostname\(false\)
X509TrustManager                          # custom implementation that accepts everything
new Random\(\).*(token|secret|password|key)
catch\s*\(\s*Exception\s+\w+\s*\)\s*\{\s*\}
```

**Go**
```
exec\.Command\(.*\+
InsecureSkipVerify:\s*true
os\.Getenv\(.*\)\s*//\s*default
math/rand                                 # instead of crypto/rand for security use
```

**Terraform / infrastructure**
```
0\.0\.0\.0/0                               # review if on a sensitive port (22, 3306, 5432, 6379...)
"Action"\s*:\s*"\*"
"Resource"\s*:\s*"\*"
sensitive\s*=\s*false                      # on a variable/output that looks like a secret
:latest                                    # unpinned image tag
ARG\s+.*_(PASSWORD|SECRET|KEY|TOKEN)
ENV\s+.*_(PASSWORD|SECRET|KEY|TOKEN)
runAsRoot:\s*true
allowPrivilegeEscalation:\s*true
hostNetwork:\s*true
```

**CI/CD**
```
uses:\s*[^@]+@main
uses:\s*[^@]+@master
uses:\s*[^@]+@latest
2>/dev/null(?!.*\|\|)                     # possible error swallowing under set -eo pipefail
```

## Report template

```markdown
---
diff_base: "<base>...<head resolved to a commit hash>"
reviewed_at: "<YYYY-MM-DD HH:MM:SS>"
files_reviewed:
  - "<path/file1>"
  - "<path/file2>"
findings_count:
  critical: 0
  high: 0
  medium: 0
  low: 0
  info: 0
escalate_to_security_team: false
status: "<changes-needed | approved-with-remarks | approved-no-remarks>"
---

# Security Review — <slug/branch/PR>

## Findings

<!-- One block per real finding. Remove this example block and repeat for each vulnerability found. -->

### 🔴 CRITICAL — <Vulnerability title>

**File:** `path/to/file.ext`, line X
**Category:** <e.g. SQL Injection, IDOR, Hardcoded Secret>
**CWE:** CWE-XXX | **OWASP:** A0X

**Problem:**
<What's wrong and why it's a risk in this specific context.>

**Impact:**
<What an attacker can actually do — specific to this application, not generic.>

**Fix:**
```<language>
// ready-to-apply fixed code
```

**Reference:** <relevant OWASP / CWE / CVE>

<!-- end of example block -->

## What was reviewed and is adequate

<Briefly list the areas/files reviewed with no findings — don't silently omit, confirm they were checked.>

## Summary

### Findings table

| Severity | Category | File | Line |
|---|---|---|---|
| 🔴 | | | |

### Top 3 priorities

1.
2.
3.

### What couldn't be analyzed

<External context dependencies, missing files, tool unavailable in the environment, etc. If none, state "Nothing to report".>

### Escalation recommendation

<Yes/No + justification, per the criteria below.>
```

## Escalation to the security team

Explicitly recommend escalation when you find:
- Any 🔴 CRITICAL finding.
- Suspicion of a compromised dependency (supply chain — suspicious name, unexpected `postinstall`, recently changed maintainer).
- A vulnerability with uncertain impact that requires pentest validation.
- Production data potentially exposed or logged inappropriately.

## Limits and cautions

- Never invent a project structure or convention that doesn't exist — discover it by actual inspection (Steps 1–2).
- Real tools (SAST/SCA/tfsec/checkov/audit) are always preferable to inference by reading, when available in the environment — don't skip this for convenience.
- This skill complements human review, pentesting, and the repository's already-configured automated scanners — it is not a guarantee of absence of vulnerabilities.
- Don't generate artificial findings just to fill sections; if the code is adequate, say so clearly.
- The file under `.ai/security-reviews/` is the durable artifact (can be versioned or attached to the PR); the chat conversation is just the interactive surface.
