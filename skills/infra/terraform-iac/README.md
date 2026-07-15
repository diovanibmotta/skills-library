# terraform-iac

Applies corporate-grade conventions for writing, organizing, and importing Terraform code — self-explanatory singular file naming, state isolation per layer/component/cloud-provider, module documentation, and mandatory validation commands.

## Trigger

```
/terraform-iac
```

## What it does

1. Checks whether the current repository already documents its own Terraform conventions (`CLAUDE.md`/`AGENTS.md`/`README.md`/`CONTRIBUTING.md`) and defers to them when they conflict with this skill's defaults
2. Applies singular, self-explanatory file naming (`provider.tf`, `backend.tf`, `variable.tf`, `output.tf`, `locals.tf`, `main.tf`, `data.tf`, `check.tf`)
3. Keeps English-only `description` fields in code regardless of the repository's documentation language
4. Enforces state isolation per `<layer>/<component>/<cloud-provider>/` directory structure
5. Places scripts, keypairs, and secrets in dedicated subdirectories, excluded from version control
6. Requires a module `README.md` (definition, resources created, prerequisites, how to run, mermaid diagram when relevant)
7. Uses `locals` for redundant values, `check` blocks for post-provisioning invariants, and `terraform_data` for reconfiguration triggers
8. Walks through importing pre-existing (ClickOps or foreign-state) resources with generated code + README
9. Prefers official Terraform Registry modules over hand-written resources
10. Uses `dynamic`/`for_each` to avoid repetition, and `sensitive = true` on every credential/secret
11. Runs `terraform init`/`fmt`/`validate` at the end of every change and self-resolves resulting errors

## Target stack

Any cloud provider (AWS, Azure, GCP, ...) and any Terraform repository — module structure conventions, not provider-specific resources.

## Usage

```
/terraform-iac
```

Invoke while creating, changing, or importing a Terraform module/component.

## Notes

- Existing, documented repository conventions always take precedence over this skill's defaults — it only fills in what isn't already established.
- Does not replace `terraform plan`/`apply` review discipline or provider-specific security scanners (`tfsec`/`checkov`/`tflint`) — it standardizes structure and hygiene around them.
