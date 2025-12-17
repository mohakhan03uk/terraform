
# HCP Terraform — Concise Guide

## Overview
**HCP Terraform** (formerly Terraform Cloud) manages Terraform runs in a consistent, reliable, and secure environment. It provides:
- Role-based access control (RBAC)
- Private Module Registry (share and version modules & providers)
- Policy Enforcement (Sentinel / OPA integrations)
- Remote State Management with locking and versioning
- VCS Integrations (GitHub, GitLab, Azure Repos, Bitbucket)
- Run automation via **CLI**, **VCS**, or **API**

> Not everything is free — HCP Terraform offers Free and paid plans (Standard, Plus, Enterprise) with progressively advanced features.

---

## Core Structure

### 1) Organizations
- Shared space for teams to collaborate on **workspaces**.
- **Billing and plans** are managed at the **organization** level.
- A user can belong to **multiple organizations**, each with different billing plans.

### 2) Workspaces
- The fundamental unit for managing infrastructure and **Terraform state**.
- Each workspace corresponds to a **state**; do not upload `.tf` files directly.
- Typically **connected to a VCS repository** to fetch code, or linked via the **cloud block** in CLI/API-driven workflows.

**Workspace & Configuration Files**
- You don’t upload `sample.tf` to the workspace directly.
- Connect the workspace to a **Git repository** (for VCS-driven) or link your **local directory** via the `cloud` block (for CLI/API-driven).

**Workspace vs Directories**
- **Workspace** = Cloud-managed state, runs, permissions, policies, variables.
- **Directory** = Local folder on your machine containing Terraform configuration.

### 3) Projects
- Logical groupings of **workspaces** for organization (by team, BU, service, environment, etc.).
- In **Standard Edition+**, you can grant team access at the **project** level to manage groups of workspaces.

---

## Run Workflow Modes (When Creating a Workspace)

1. **CLI-driven workflow**
   - You run `terraform plan` / `terraform apply` **locally**.
   - **State**, logs, and run history are stored in **HCP Terraform**.
   - Your local working directory is **linked** to an HCP Workspace.

2. **VCS-driven workflow**
   - Runs are triggered automatically by **commits** / **pull requests** in the connected repo.
   - Ideal for GitOps and policy checks in CI on PRs.

3. **API-driven workflow**
   - Runs are triggered via **API** (great for CI/CD pipelines like GitHub Actions, Azure DevOps, Jenkins, etc.).

---

## CLI-driven Run Workflow — Practical Example (Safe & Local)

### Goal
Run locally, store state and run history in HCP Terraform, and create a local file to avoid cloud creds.

### Prerequisites
- HCP Terraform account and an **Organization** (e.g., `acme-org`)
- A **Workspace** (CLI-driven) named `demo-workspace` (or created automatically by the CLI)
- Terraform CLI installed (v1.1+ recommended)

### Steps

#### 1) Create/Confirm Workspace (CLI-driven)
- In HCP Terraform → **Workspaces → New → CLI-driven** → name: `demo-workspace`.
- (Optional) assign to a **Project**.

#### 2) Authenticate the CLI
```bash
terraform login
# or set a token environment variable (Linux/macOS)
export TF_TOKEN_app_terraform_io=<your-token>
# or on Windows PowerShell
setx TF_TOKEN_app_terraform_io "<your-token>"
```
##### 🔍 terraform login — What It Needs
- Confirms hostname (default: `app.terraform.io`)
- Opens browser for HCP login
- Generates token → paste back into CLI
- Stores token in `~/.terraform.d/credentials.tfrc.json`
- **Organization name is NOT asked during login** — it’s specified later in `main.tf`:

#### 3) Link local dir to HCP Workspace
Create folder `hcp-cli-demo` → `main.tf`:
```hcl
terraform {
  cloud {
    organization = "acme-org"     # <-- replace with your org
    workspaces {
      name = "demo-workspace"     # <-- must match
    }
  }

  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

# Generate a random pet name
resource "random_pet" "name" {
  length = 2
}

# Create a local text file with the random name
resource "local_file" "demo" {
  filename = "${path.module}/hello_${random_pet.name.id}.txt"
  content  = "Hello from CLI-driven HCP Terraform! Name = ${random_pet.name.id}\n"
}

# Output for quick confirmation
output "file_created" {
  value = local_file.demo.filename
}
```

#### 4) Initialize, Plan, Apply (locally)
```bash
terraform init
terraform plan
terraform apply -auto-approve
```
- You run everything **from your terminal**.
- **State** and run history are stored **remotely** in HCP Terraform.
- A file `hello_<random>.txt` appears locally.

#### 5) Verify in HCP UI
- Org → Workspaces → `demo-workspace`
- Check **Runs**, **State versions**, **Outputs**, **Logs**.

#### 6) Update & Re-run (optional)
Modify the content to include a timestamp:
```hcl
resource "local_file" "demo" {
  filename = "${path.module}/hello_${random_pet.name.id}.txt"
  content  = "Updated at ${timestamp()}. Name = ${random_pet.name.id}\n"
}
```
Then re-run:
```bash
terraform plan
terraform apply -auto-approve
```

#### 7) Clean up
```bash
terraform destroy -auto-approve
```
Optionally delete the workspace in HCP to remove the remote state.

---

## Variables & Sensitive Values (CLI-driven)
- Use the **Variables** tab in the HCP workspace to set Terraform variables (mark sensitive ones as **Sensitive**).
- Or use local CLI options `-var`, `-var-file`, or `terraform.tfvars`.

**Example**
```hcl
variable "environment" { type = string }
output "env" { value = var.environment }
```
`terraform.tfvars`:
```hcl
environment = "dev"
```

---

## Tips & Tricks
- **Use Projects for RBAC**: Group workspaces by team/service/environment and assign team access at the **project** level (Standard+ plans).
- **Naming Conventions**: Standardize workspace names (e.g., `svc-env-region`) for clarity in policies and automation.
- **State Safety**: Let HCP manage locking/versioning; avoid local state files for team work.
- **Policy-as-Code**: Start with **Sentinel soft-mandatory** policies (advisory) before enforcing hard gates.
- **Run Tasks**: Integrate security/compliance scanners as pre-/post-plan tasks.
- **Variable Sets**: Share common variables (e.g., tags, cost center) across multiple workspaces.
- **Workspace Tags**: Tag workspaces to enable filtered views, automation targeting, and reporting.
- **Agents (for private VCS/repos)**: Use HCP Terraform Agents when your code or providers must run inside your network.
- **Ephemeral Envs**: Use **speculative plans** on PRs (VCS-driven) to preview changes without affecting state.
- **Outputs for Pipelines**: Surface essential outputs for downstream jobs (store as workspace outputs or use the API to fetch).
- **Provider & Module Version Pinning**: Use conservative semver ranges (e.g., `~>`), and a private registry for reviewed modules.
- **Workspaces ≠ Environments**: Use **one workspace per environment** rather than a single workspace with many variables.
- **Drift Detection**: Schedule or automate `plan` to detect drift and notify teams.
- **Access Tokens**: Prefer **team or workspace tokens** over personal tokens in automation.

---

## Quick Reference Commands
```bash
# Login and link to HCP
terraform login

# Initialize project (downloads providers, sets up cloud backend)
terraform init

# Create an execution plan
terraform plan

# Apply the plan automatically
terraform apply -auto-approve

# Destroy managed resources
terraform destroy -auto-approve
```

---

## MCQs — Check Your Understanding

**1. In HCP Terraform, which of the following is true about Workspaces?**
A) They are local directories on your machine.  
B) They each maintain a separate remote state and run history.  
C) They can only be used with VCS-driven workflows.  
D) They store `.tf` files uploaded directly via the UI.  
**Answer:** B

**2. In a CLI-driven workflow, where do `plan`/`apply` execute and where is state stored?**
A) Execute in HCP; state stored locally.  
B) Execute locally; state stored in HCP.  
C) Execute and state both in HCP.  
D) Execute and state both locally.  
**Answer:** B

**3. What’s the recommended way to connect a local directory to an HCP Workspace for CLI-driven runs?**
A) Upload `.tf` files to the workspace.  
B) Use the `cloud` block in `terraform` configuration.  
C) Use a `.backend` file.  
D) You cannot connect a local directory.  
**Answer:** B

**4. Which feature helps enforce organizational rules (e.g., approved regions, tags)?**
A) Variable Sets  
B) Sentinel / Policy-as-Code  
C) Private Registry  
D) Workspace Tags  
**Answer:** B

**5. Projects in HCP Terraform are primarily used to:**
A) Run Terraform commands.  
B) Organize workspaces and apply RBAC at a group level.  
C) Store Terraform state files.  
D) Replace organizations.  
**Answer:** B

**6. For shared variables (e.g., cost center tags) across multiple workspaces, use:**
A) Workspace Outputs  
B) Variable Sets  
C) CLI `-var` flags only  
D) Provider meta-arguments  
**Answer:** B

**7. The best practice for environments (dev/test/prod) is to:**
A) One workspace for all environments with many variables.  
B) One workspace per environment.  
C) One workspace per developer.  
D) Use only local state.  
**Answer:** B

**8. Which workflow is best for PR-based reviews and automated checks?**
A) CLI-driven  
B) VCS-driven  
C) API-driven  
D) Manual-only  
**Answer:** B

**9. If your providers or VCS need to run inside your private network, you should use:**
A) HCP Terraform Agents  
B) Local-only state  
C) Workspace tokens  
D) Terraform Cloud Destroy  
**Answer:** A

**10. When pinning module/provider versions, a safe semver strategy is:**
A) Never pin versions.  
B) Use exact versions only.  
C) Use `~>` to allow non-breaking updates.  
D) Always track `latest`.  
**Answer:** C

---

## Appendix — Sample `cloud` Block Patterns

**Single named workspace**
```hcl
terraform {
  cloud {
    organization = "acme-org"
    workspaces { name = "demo-workspace" }
  }
}
```

**Workspace selection by prefix (advanced)**
```hcl
terraform {
  cloud {
    organization = "acme-org"
    workspaces { prefix = "svc-" }
  }
}
# The actual workspace is selected via env var TF_WORKSPACE or `terraform workspace select`
```

---
