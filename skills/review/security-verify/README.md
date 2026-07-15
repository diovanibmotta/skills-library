# security-verify

Runs a structured security review over the code changes of a branch/PR/diff — language-, stack-, and cloud-provider-agnostic (application code, infrastructure as code, containers, CI/CD). Classifies findings by severity, maps them to OWASP Top 10/CWE, and delivers ready-to-apply fixes.

## Trigger

```
/security-verify
```

## What it does

1. Determines the diff scope (asks for the comparison base when it isn't obvious from context)
2. Detects which stacks are touched (application code, dependency manifests, infrastructure/containers/CI) and loads the matching checklist
3. Runs a quick pattern scan (secrets, `eval`/`exec`, insecure deserialization, disabled TLS verification, etc.) as a first filter
4. Performs a manual, contextual review covering secrets/config, input validation, injection, authentication/authorization (including IDOR), sensitive data exposure/cryptography, error handling/logging, and rate limiting
5. Reviews dependencies and infrastructure when the diff touches them, running real scanners (`npm audit`, `pip-audit`, `tfsec`, `checkov`, etc.) when available in the environment
6. Classifies every finding on a 🔴 CRITICAL → ⚪ INFO scale, mapped to CWE and OWASP Top 10 (2021)
7. Generates a versioned report at `.ai/security-reviews/<slug>/review-<timestamp>.md` with front matter tracking the diff base and finding counts
8. Presents a chat summary: findings table, top 3 priorities, what couldn't be analyzed, and an escalation recommendation when warranted

## Target stack

Any language, any stack, any cloud provider — application code (Java, Python, TypeScript/JS, Go, ...), containers, Kubernetes, CI/CD pipelines, and Terraform.

## Usage

```
/security-verify
/security-verify against main
```

## Notes

- Complements (does not replace) automated SAST/SCA/IaC scanners already configured in the repository — it runs them when available and folds the real result into findings.
- Repository-specific security conventions (`CLAUDE.md`/`AGENTS.md`/`SECURITY.md`) take precedence over this skill's generic checklists when they conflict.
- The report file under `.ai/security-reviews/` is the durable artifact — meant to be versioned or attached to the PR.
