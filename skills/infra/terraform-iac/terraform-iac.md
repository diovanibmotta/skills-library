---
description: Applies corporate-grade conventions for writing, organizing, and importing Terraform (IaC) code — self-explanatory singular file naming (variable.tf, output.tf, provider.tf, main.tf, backend.tf, locals.tf), English-only descriptions in code, state isolation per layer/component/cloud-provider, scripts/keypairs/secrets structure, module README.md (with a mermaid diagram when it helps), terraform workspace per environment, check blocks for post-provisioning validation, terraform_data for reconfiguration triggers, importing pre-existing resources (ClickOps or another Terraform state) with generated code+README, locals for redundant values, preferring official Terraform Registry modules, dynamic/for_each to avoid repetition, sensitive=true on credentials/secrets, and mandatory terraform init/fmt/validate at the end. Use whenever the user asks to create, change, import, or review a Terraform module/component, scaffold infrastructure as code from scratch, or work in any IaC repository (AWS, Azure, GCP, etc.).
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, AskUserQuestion
---

# Terraform IaC — Corporate Conventions

Cloud-provider-agnostic and repository-agnostic skill (AWS, Azure, GCP, ...) — applies to any Terraform project. The rules below are the house standard for **new** modules/components.

## Precedence: existing local conventions win over this skill

Before applying any rule from this skill, check whether the current repository already documents its own conventions (`CLAUDE.md`, `AGENTS.md`, `README.md`, `CONTRIBUTING.md`). If the repository already has an established, documented pattern that conflicts with this skill (e.g. plural file names — `variables.tf`/`outputs.tf`/`providers.tf` — or a state-key structure different from the layer/component/cloud-provider one described here), **follow the repository's already-documented pattern** — do not restructure an existing repo to match this skill unless the user explicitly asks for that. This skill defines the standard for:

- New repositories/modules with no established convention yet.
- Cases where the user explicitly asks to apply/migrate to these conventions.

When you do apply this skill's convention over a repo default (or vice versa), state clearly which one you followed and why (e.g. "used plural `variables.tf` because that's already the documented standard in this repo's CLAUDE.md, instead of this skill's singular default").

## 1. File naming

Every `.tf` file should have a self-explanatory, singular name reflecting its single purpose. Never mix unrelated concerns into one generic file (`main.tf` grows unbounded) when a more specific name would fit.

| File | Content |
|---|---|
| `provider.tf` | `terraform {}` block (version pins) + `provider` block(s) |
| `backend.tf` | `terraform { backend "..." {} }` block |
| `variable.tf` | all `variable` declarations for the module |
| `output.tf` | all `output` declarations for the module |
| `locals.tf` | `locals` block (derived/redundant values — see section 6) |
| `main.tf` | the module/component's primary resources |
| `data.tf` | `data` blocks (once numerous enough to clutter `main.tf`) |
| `check.tf` | `check` blocks (see section 7) |

If a module has multiple clearly distinct resource groups, prefer splitting `main.tf` into files named by resource/domain (e.g. `ec2.tf`, `s3.tf`) instead of one monolithic `main.tf` — always prioritizing a name that explains the content without opening the file.

## 2. Language

- **Descriptions inside Terraform code** (`description` of `variable`, `output`, explanatory comments) must always be **in English**, regardless of the repository's dominant documentation language.
- Resource/variable/output/local names: English, `snake_case`.
- `README.md`, commit messages, and other documentation follow whatever language the repository already established — only the description *inside* `.tf` code is always English.

## 3. State isolation per layer/component/cloud-provider

Every module/component within a solution must have its **own isolated state**, never shared with another component. The directory structure mirrors the state key 1:1:

```
<layer>/<component>/<cloud-provider>/
```

Examples:

```
infra/users/aws/            -> state key: infra/users/aws/terraform.tfstate
shared/ai-gateway/aws/      -> state key: shared/ai-gateway/aws/terraform.tfstate
```

`<layer>` is the logical grouping (e.g. `infra`, `shared`, a team/product name), `<component>` is the specific module/solution (e.g. `users`, `ai-gateway`), `<cloud-provider>` is that component's provider (`aws`, `azure`, `gcp`) — the same component can have parallel implementations across more than one provider, each with its own state.

Never point two components at the same state key, and never have one component read/write another's state directly — use `terraform_remote_state` as a data source, or explicit outputs, when values need to be shared across components.

## 4. Scripts, keypairs, and secrets

Whenever a module needs helper scripts (bootstrap, user-data, deploy hooks), create them under:

```
<layer>/<component>/<cloud-provider>/scripts/
```

Keypair or secret files **generated/consumed** by the module (e.g. an `aws_key_pair` private key, a `.tfvars` with a password) go into dedicated subdirectories, named after the Terraform resource that references them:

```
<layer>/<component>/<cloud-provider>/keypairs/<resource_terraform_name>.<ext>
<layer>/<component>/<cloud-provider>/secrets/<resource_terraform_name>.<ext>
```

These directories should almost always be in the repository's `.gitignore` — check before generating real files in them; never commit sensitive material in plain text.

## 5. Module README.md

Every module/component (`<layer>/<component>/<cloud-provider>/README.md`) must contain, at minimum:

1. **Definition** — what this module is and why it exists.
2. **Resources created** — an objective list of provisioned resources.
3. **Prerequisites** — what must exist/be configured beforehand (credentials, variables, backend, dependencies on other components).
4. **How to run** — commands (`terraform init/plan/apply`) and required variables.
5. **Mermaid diagram** (when the module has more than one resource communicating with each other, or relevant external dependencies) — showing the flow/communication between the created resources.

## 6. `locals` for redundant values

Whenever a value repeats in more than one place inside the module (a base name for tags, a naming prefix, an ARN built from other values, etc.), extract it into `locals.tf`. Don't create `locals` for values used only once.

## 7. `check` blocks for post-provisioning validation

Whenever it makes sense to guarantee an invariant after a resource is created/changed, use a `check` block that emits a warning (not a fatal error) if the condition isn't met. Example — ensuring an instance's SSH port is reachable:

```hcl
check "ssh_port_open" {
  data "aws_security_group" "this" {
    id = aws_security_group.instance.id
  }

  assert {
    condition = anytrue([
      for rule in data.aws_security_group.this.ingress : rule.from_port <= 22 && rule.to_port >= 22
    ])
    error_message = "Security group ${aws_security_group.instance.id} does not allow inbound SSH (port 22)."
  }
}
```

Use this for security, connectivity, or operational pre-condition invariants — not for input validation (that's `variable { validation {} }`'s job).

## 8. `terraform_data` for reconfiguration triggers

When a change to a resource needs to trigger an external action (re-provision a script, restart a service, invalidate a cache) every time the resource is created/changed/destroyed, use `terraform_data` with `triggers_replace` and a `provisioner` (preferably `local-exec`/`remote-exec`) — don't use the legacy `null_resource` in new code.

```hcl
resource "terraform_data" "reload_config" {
  triggers_replace = [aws_instance.this.id, aws_instance.this.ami]

  provisioner "local-exec" {
    command = "./scripts/reload-config.sh ${aws_instance.this.id}"
  }
}
```

## 9. Importing pre-existing resources

When the user asks to import a resource that already exists in the cloud (created by another Terraform project, another state, or ClickOps):

1. Write the corresponding Terraform resource block (using `terraform plan -generate-config-out` when available, or writing it manually + an `import {}` block / `terraform import`).
2. Run the import and confirm with `terraform plan` that there's no diff (or that the remaining diff is intentional and justifiable).
3. Adjust the generated code to follow every convention in this skill (file names, `locals`, `sensitive`, etc. — imported code is rarely clean out of the box).
4. Create/update the module's `README.md` documenting that this resource was imported, where it came from, and why.

Never leave an imported resource undocumented — it's the only way whoever reads the module later understands that resource wasn't born there.

## 10. Prefer official Terraform Registry modules

Before writing a resource from scratch, check whether an official/verified module on the Terraform Registry already covers the need (e.g. `terraform-aws-modules/vpc/aws`, `terraform-aws-modules/eks/aws`). Prefer reusing these modules over reimplementing the same logic by hand, unless the official module brings unnecessary complexity/resources for the use case — in that case, justify the decision not to use it (a short comment or a note in the README).

## 11. `dynamic` and `for_each`

- Use `dynamic` blocks to avoid repeating identical nested blocks (e.g. multiple `ingress`/`egress` rules in a security group, multiple `lifecycle_rule` in a bucket).
- Use `for_each` (not `count`) to iterate over lists/maps of elements with their own identity, and to externalize configuration (e.g. a list of users, environments, or instances coming from a variable). Reserve `count` for homogeneous replicas with no relevant individual identity.

## 12. Sensitive data

Every variable or output that represents a credential, password, token, or secret handled/generated by Terraform must have `sensitive = true`. This holds even when the value also goes into a `keypairs/`/`secrets/` file (section 4) — the two protections are complementary, one doesn't replace the other.

## 13. `terraform workspace` for environments

Use `terraform workspace` to manage distinct environments of the same component (`develop`, `homolog`, `production`) when the repository doesn't already use its own mechanism (e.g. per-environment directories/pipelines). Don't mix both mechanisms in the same component — check which environment strategy the repository already follows (see "Precedence" above) before introducing workspaces where another convention already exists.

## 14. Workflow

**New module/component:**
1. Create the directory structure `<layer>/<component>/<cloud-provider>/` with the files from section 1.
2. Write the code following sections 2–13 as applicable.
3. Write the `README.md` (section 5).
4. Run `terraform init`.
5. Run `terraform fmt` and `terraform validate`.

**Change to an existing module:**
1. Check the conventions already in use in the module/repository ("Precedence" section) and follow them.
2. Apply the change.
3. Run `terraform fmt` and `terraform validate` when done.

**Resource import:** follow section 9, then `terraform fmt`/`terraform validate` as usual.

## 15. Mandatory commands and error resolution

- `terraform init` — whenever creating a new module/component (or when the backend/providers change).
- `terraform fmt` — always when done coding, before considering the work finished.
- `terraform validate` — always when done coding.
- If `fmt`/`validate` (or `tflint`/`tfsec`/`checkov`, when the repository uses them) report an error: analyze the cause and fix it yourself. Only ask for human help if, after investigating, the error persists or requires a decision only the user can make (e.g. a missing credential, an architecture decision).

## Limits and cautions

- Never invent a directory/naming convention that doesn't already exist in the repository without checking "Precedence" first.
- Don't restructure an existing repo to match this skill's defaults unless explicitly asked.
- Don't skip `terraform fmt`/`terraform validate` for convenience — they're the minimum correctness gate this skill guarantees.

## Final checklist before considering the module done

- [ ] Files named self-explanatorily, following the repository's convention ("Precedence" section) or this skill's singular default.
- [ ] Every `variable`/`output` `description` in English.
- [ ] State isolated per `<layer>/<component>/<cloud-provider>` (or the repo's existing convention).
- [ ] Scripts/keypairs/secrets in the correct subdirectories, secrets excluded from version control.
- [ ] `README.md` present and complete (with a mermaid diagram if applicable).
- [ ] Redundant values extracted into `locals`.
- [ ] `check` blocks for relevant post-provisioning invariants.
- [ ] `terraform_data` for reconfiguration triggers, when applicable.
- [ ] Official Registry modules considered before hand-written resources.
- [ ] `dynamic`/`for_each` used to avoid repetition.
- [ ] `sensitive = true` on every credential/secret.
- [ ] `terraform init` (if new) + `terraform fmt` + `terraform validate` run without error.
