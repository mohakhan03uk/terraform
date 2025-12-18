
# HCP Terraform Workspaces vs. CLI Workspaces

This guide explains **how workspaces work in HCP Terraform (Terraform Cloud/Enterprise)** and how they differ from **CLI workspaces**. It also provides configuration examples, environment strategies, tips, and MCQs.

---
## 1) Conceptual Difference

- **CLI Workspaces (local/remote backends like S3):**
  - Multiple state files for the **same root module** managed via `terraform workspace ...` commands.
  - Live under `terraform.tfstate.d/<workspace>/` (local backend) or under a workspace-aware key/prefix in remote backends.

- **HCP Terraform Workspaces:**
  - A **first-class unit** managed by HCP Terraform: each workspace has **its own runs, state, variables, policies, and settings**.
  - Workspaces are created and managed via **HCP UI, API, or VCS integrations** (not via CLI workspace commands).

> Treat HCP Terraform workspaces as **environment units** (e.g., `app-dev`, `app-stage`, `app-prod`) rather than state partitions of a single directory.

---
## 2) Can I use these CLI commands in HCP Terraform?

```
terraform workspace list
terraform workspace show
terraform workspace new kplabs
terraform workspace select prod
terraform workspace delete stage
```

**Short answer:** Not for managing HCP Terraform workspaces.

- When using the `cloud` backend (HCP Terraform), the CLI **workspace** feature is effectively **disabled**. The commands above **do not create/select/delete HCP workspaces**.
- You typically see only `default` if you try `terraform workspace list` while using the HCP backend, because HCP Terraform **doesn't use CLI workspaces**.

**Do this instead:** Create/select workspaces **in HCP Terraform** (UI/API) and point your config to a specific HCP workspace via the `cloud` backend block.

---
## 3) HCP Terraform Backend Configuration

Use the `cloud` block inside `terraform {}` to target your organization and workspace:

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "app-prod"    # Target ONE workspace by name
      # or
      # tags = ["service:web", "env:prod"]  # Target by tags (Terraform Cloud/E) when applicable
    }
  }
}
```

**Notes:**
- The `cloud` block replaces backend config like `s3`/`azurerm`.
- Authentication to HCP Terraform is done via `terraform login` and storing a token in the CLI credentials.
- Terraform Cloud requires exactly one matching workspace
   - 0 or >1 matches → ❌ hard error
   - Terraform never auto-selects or iterates
- If you want to run Terraform for multiple workspaces, you must:
    - run Terraform once per workspace , usually via a CI loop or matrix job

---
## 4) Recommended Environment Strategy in HCP Terraform

- **One HCP workspace per environment** per stack, e.g.:
  - `platform-network-dev`
  - `platform-network-stage`
  - `platform-network-prod`
- Configure each workspace with its own **variables** (Terraform variables and environment variables), **run triggers**, and **policy sets** (Sentinel/OPA if applicable).
- Drive runs via **VCS branch per environment** (e.g., `main` → prod, `develop` → dev) or via **CLI-driven runs** from CI with the `cloud` block targeting the correct workspace.

---
## 5) Using `terraform.workspace` inside code

Even with HCP Terraform, you can still reference `terraform.workspace` **inside your HCL**. However:
- In HCP Terraform, the active workspace is the **HCP workspace name** (e.g., `app-prod`).
- Use this primarily for **tags and naming**, not as your main isolation strategy.

Example:
```hcl
locals {
  tags = {
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket" "logs" {
  bucket = "platform-logs-${terraform.workspace}"
  tags   = local.tags
}
```

---
## 6) Practical Workflow (CLI-Driven in HCP Terraform)

1. **Log in to HCP Terraform**:
   ```bash
   terraform login
   ```
2. **Define the HCP workspace in code** (cloud block as above).
3. **Initialize**:
   ```bash
   terraform init
   ```
4. **Plan/Apply** (variables are managed in the HCP workspace UI or supplied via `-var-file` if allowed by your workflow):
   ```bash
   terraform plan
   terraform apply
   ```

> You **do not** run `terraform workspace select prod`. Instead, the `cloud` block already pins the run to the `app-prod` HCP workspace.

---
## 7) Managing Workspaces (HCP UI/API)

- **Create** a workspace: HCP UI → Workspaces → New → connect repo or CLI workflow.
- **Variables**: Set Terraform variables and environment variables per workspace (e.g., AWS credentials, region, instance sizes).
- **Policy sets**: Attach org-level or workspace-level policies.
- **Run triggers**: Set dependency relationships between workspaces.
- **VCS settings**: Connect a GitHub/GitLab repo and branch, choose speculative vs plan/apply triggers.

---
## 8) CI/CD: Selecting the Workspace

With CI (GitHub Actions/Azure DevOps), prefer **one pipeline per environment workspace** or pass a parameter to pick the HCP workspace. Example pseudo-step:

```bash
# Example: set TF_CLOUD_ORGANIZATION and choose workspace name dynamically
export TF_CLOUD_ORGANIZATION=my-org
export TF_WORKSPACE=app-stage

# Option A: cloud block hardcodes name; keep separate jobs per env
# Option B: generate a small backend partial file per env

terraform init
terraform plan
terraform apply
```

> Ensure your pipeline uses the correct **HCP workspace name**. Avoid mixing CLI workspace commands.

---
## 9) Common Pitfalls & Best Practices

- **Pitfall: Mixing concepts.** Avoid using `terraform workspace new/select` with HCP. They manage different types of workspaces.
- **Strong isolation:** Use separate HCP workspaces, and often separate **cloud accounts/projects** for prod vs dev.
- **Variables security:** Store sensitive values in HCP Terraform **sensitive variables** or use a secrets manager.
- **Tags & naming:** Use `terraform.workspace` for resource naming consistency across environments.
- **Modules:** Parameterize modules with variables; do not bake environment logic deep inside modules.
- **Drift & visibility:** Use HCP run history, cost estimation, and policies to monitor and control changes.

---
## 10) Quick Reference

- Authenticate: `terraform login`
- Target HCP workspace: `terraform { cloud { organization = "<org>"; workspaces { name = "<ws>" } } }`
- Initialize: `terraform init`
- Run: `terraform plan` / `terraform apply`
- Manage workspaces: **HCP UI** or **API**, not CLI workspace commands

---
## 11) MCQs (with answers)

1. **In HCP Terraform, how are workspaces managed?**
   - A) Via `terraform workspace new/select/delete`
   - B) Via HCP UI/API ✅
   - C) Only via VCS branches
   - D) They are created automatically by `terraform init`

2. **What does the `cloud` block do?**
   - A) Configures local backend
   - B) Configures remote state in S3
   - C) Targets an HCP Terraform organization/workspace ✅
   - D) Sets provider versions

3. **When using HCP Terraform, `terraform workspace list` typically shows:**
   - A) All HCP workspaces
   - B) Only CLI workspaces in the local backend
   - C) Usually just `default`, because HCP doesn’t use CLI workspaces ✅
   - D) Nothing

4. **Best practice for environments in HCP Terraform:**
   - A) One HCP workspace per environment ✅
   - B) One HCP workspace for all environments
   - C) Only use CLI workspaces
   - D) Duplicate code repositories per environment only

5. **To run against `app-prod` in HCP Terraform, you should:**
   - A) `terraform workspace select app-prod`
   - B) Set the `cloud` block with `workspaces.name = "app-prod"` ✅
   - C) Use `-backend-config` with `workspace=app-prod`
   - D) Rename `default` workspace locally to `app-prod`

---
## 12) Summary

- HCP Terraform workspaces are **managed by HCP**, not via CLI workspace commands.
- Pin your runs to a specific HCP workspace using the **`cloud` backend block**.
- Use **one HCP workspace per environment**, manage variables and policies in HCP, and keep CLI workspace commands out of your HCP workflows.
