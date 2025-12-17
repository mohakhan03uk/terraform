
# Terraform Workspaces: A Practical Guide

Terraform workspaces let you manage **multiple, distinct state files** from the **same configuration**, enabling separate deployments (e.g., dev, stage, prod) without duplicating code.

---
## 1) Core Concepts

### Configuration vs. State
- **Configuration**: Your `.tf` files that describe resources.
- **State**: Terraform’s record of what’s been applied in an environment.
- **Workspace**: A named, logical partition of state for a single configuration directory.

> One configuration, many workspaces → many state files. Each workspace maintains its own `terraform.tfstate`. In the local backend, per-workspace state lives under `./terraform.tfstate.d/<workspace>/terraform.tfstate`.

### Why Workspaces?
- Manage **multiple environments** (dev/stage/prod) from **one codebase**.
- Keep **state isolated** per workspace.
- Parameterize behavior using `terraform.workspace` to vary values.

> Note: Workspaces are **not** a hard-security isolation boundary. For stronger isolation, prefer **separate backends**, directories, and/or separate CI jobs.

---
## 2) Workspace Lifecycle Commands

```bash
# List workspaces
terraform workspace list

# Show current workspace
terraform workspace show

# Create a new workspace
terraform workspace new kplabs

# Switch workspace
terraform workspace select prod

# Delete (only allowed for non-current workspace)
terraform workspace delete stage
```

### Typical Flow
1. `terraform init` (initialize backend/plugins).
2. `terraform workspace new dev` (first time) or `terraform workspace select dev`.
3. `terraform plan -var-file=env/dev.tfvars`.
4. `terraform apply -var-file=env/dev.tfvars`.

Repeat steps 2–4 for `stage`, `prod`, etc.

---
## 3) Using `terraform.workspace` in Code

You can branch logic or lookup values by workspace:

```hcl
locals {
  instance_types = {
    default = "t3.micro"
    dev     = "t3.micro"
    stage   = "t3.small"
    prod    = "m5.large"
  }

  tags = {
    Environment = terraform.workspace
    Owner       = "platform-team"
  }
}

resource "aws_instance" "myec2" {
  ami           = "ami-0a12345example"
  instance_type = local.instance_types[terraform.workspace]
  tags          = local.tags
}
```

> Tip: Always provide a `default` key to avoid a lookup error if you ever use the implicit `default` workspace.

---
## 4) Variable Files Per Workspace

Structure your repo to map workspace → tfvars:

```
├── main.tf
├── variables.tf
├── env/
│   ├── dev.tfvars
│   ├── stage.tfvars
│   └── prod.tfvars
```

Example `env/prod.tfvars`:
```hcl
region         = "eu-west-2"
min_capacity   = 3
max_capacity   = 10
enable_backup  = true
```

Run:
```bash
terraform workspace select prod
terraform plan  -var-file=env/prod.tfvars
terraform apply -var-file=env/prod.tfvars
```

---
## 5) Backends and Workspace Storage

### Local Backend
- States are stored in `./terraform.tfstate.d/<workspace>/terraform.tfstate`.
- **Pros**: Simple.
- **Cons**: Not collaborative; limited locking.

### Remote Backends (Recommended for Teams)
- Examples: `s3` + DynamoDB (locking), `azurerm`, `gcs`, `remote` (Terraform Cloud/Enterprise).
- Each workspace state is stored under a distinct **key** or **prefix**. For S3:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    region         = "eu-west-2"
    dynamodb_table = "tf-locks"
    encrypt        = true
    key            = "platform/${terraform.workspace}/terraform.tfstate"
  }
}
```

> Tip: Changing backend configuration after you have state requires `terraform init -migrate-state`.

---
## 6) Workspace Gotchas & Best Practices

1. **Isolation**: Workspaces separate **state**, not **code** or **identity**. For production-grade isolation, use separate cloud accounts/projects/subscriptions and/or distinct backends.
2. **Default Workspace**: Avoid using `default` for real environments; create explicit `dev`, `stage`, `prod`.
3. **Sensitive Values**: Do not hardcode secrets. Use variables, environment variables, or secret managers.
4. **Naming Conventions**: Keep workspace names simple (`dev`, `stage`, `prod`).
5. **CI/CD**: Pin the workspace in your pipeline step, e.g. `terraform workspace select $ENVIRONMENT`.
6. **Drift**: Run periodic `terraform plan` per workspace to detect drift.
7. **Modules**: Workspaces can be used at the root; module calls should be parameterized, not workspace-aware. Pass variables down.
8. **Data Sources**: Ensure data sources use the correct region/account per workspace.
9. **State Locking**: With remote backends, enable locking to avoid concurrent applies.
10. **Tags/Labels**: Stamp resources with `Environment = terraform.workspace` for traceability.

---
## 7) Full Example: Multi-Env AWS VPC + EC2

**Files:**

`variables.tf`
```hcl
variable "region" { type = string }
variable "instance_type" { type = string }
variable "cidr_block" { type = string }
```

`main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

locals {
  tags = {
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags       = local.tags
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.this.id
  cidr_block              = cidrsubnet(var.cidr_block, 4, 0)
  map_public_ip_on_launch = true
  tags                    = local.tags
}

resource "aws_instance" "web" {
  ami           = "ami-0a12345example"
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  tags          = local.tags
}
```

`env/dev.tfvars`
```hcl
region        = "eu-west-2"
instance_type = "t3.micro"
cidr_block    = "10.10.0.0/16"
```

`env/prod.tfvars`
```hcl
region        = "eu-west-2"
instance_type = "m5.large"
cidr_block    = "10.20.0.0/16"
```

**Commands:**
```bash
terraform init
terraform workspace new dev
terraform apply -var-file=env/dev.tfvars

terraform workspace new prod
terraform apply -var-file=env/prod.tfvars
```

---
## 8) Tips & Tricks

- **Guardrails with `validation`:**
  ```hcl
  variable "instance_type" {
    type = string
    validation {
      condition     = contains(["t3.micro", "t3.small", "m5.large"], var.instance_type)
      error_message = "Unsupported instance type for this module."
    }
  }
  ```

- **Workspace-aware resource names:**
  ```hcl
  resource "aws_s3_bucket" "logs" {
    bucket = "platform-logs-${terraform.workspace}"
  }
  ```

- **Protect prod with `-target` and `-auto-approve` discipline**: Avoid `-auto-approve` in prod; review plans thoroughly. Use change management.

- **Different accounts via provider aliases:**
  ```hcl
  provider "aws" {
    alias  = "dev"
    region = "eu-west-2"
  }
  provider "aws" {
    alias  = "prod"
    region = "eu-west-2"
  }

  module "stack" {
    source  = "./modules/stack"
    providers = {
      aws = terraform.workspace == "prod" ? aws.prod : aws.dev
    }
  }
  ```

- **Backend key templating with workspaces:** As shown earlier for S3, use `key = "platform/${terraform.workspace}/terraform.tfstate"`.

---
## 9) FAQs

**Q: Can I use workspaces to deploy completely different stacks (e.g., network vs. data platform)?**
- A: Use **modules** or separate root modules. Workspaces are for separate *instances* of the same stack.

**Q: Are workspaces sufficient for production isolation?**
- A: Often **no**. Prefer separate cloud accounts/projects and backends plus role-based access control.

**Q: How do I migrate state across workspaces?**
- A: Use `terraform state` commands or `terraform init -migrate-state` when changing backends. Take backups.

**Q: What about Terraform Cloud?**
- A: Terraform Cloud has the concept of **workspaces** too, mapping to runs and state; similar idea but managed service.

---
## 10) MCQs (with answers)

1. **What does a Terraform workspace primarily isolate?**
   - A) Providers
   - B) Variables
   - C) State files ✅
   - D) Modules

2. **Where does the local backend store per-workspace state?**
   - A) `./terraform.tfstate`
   - B) `./terraform.tfstate.d/<workspace>/terraform.tfstate` ✅
   - C) `/var/lib/terraform/<workspace>.tfstate`
   - D) Remote backend bucket

3. **Which expression returns the current workspace name?**
   - A) `var.workspace`
   - B) `local.workspace`
   - C) `terraform.workspace` ✅
   - D) `path.workspace`

4. **For stronger environment isolation, you should prefer:**
   - A) Only using workspaces
   - B) Separate backends and cloud accounts/projects ✅
   - C) Using `-auto-approve` in production
   - D) Keeping all envs in the default workspace

5. **In an S3 backend, a common pattern to separate states by workspace is to set:**
   - A) `region = terraform.workspace`
   - B) `key = "stack/${terraform.workspace}/terraform.tfstate"` ✅
   - C) `bucket = terraform.workspace`
   - D) `encrypt = terraform.workspace`

6. **Which is *not* a recommended practice?**
   - A) Parameterize via tfvars per environment
   - B) Stamp resources with `Environment = terraform.workspace`
   - C) Use `default` workspace as production ✅
   - D) Enable state locking in remote backends

7. **To run different providers per workspace, you can use:**
   - A) Provider aliases with conditional mapping ✅
   - B) Modules only
   - C) `terraform.workspace` inside provider blocks without aliases
   - D) `terraform plan` without variables

8. **Deleting a workspace is allowed when:**
   - A) It’s the current workspace
   - B) The workspace has resources
   - C) It’s not the current workspace ✅ and you’ve removed resources (recommended)
   - D) You’re using any backend

9. **Which command shows the active workspace?**
   - A) `terraform show workspace`
   - B) `terraform workspace current`
   - C) `terraform workspace show` ✅
   - D) `terraform env show`

10. **Workspaces are most appropriate when:**
    - A) Deploying the same stack to multiple environments ✅
    - B) Managing entirely different stacks
    - C) Sharing a single state among all envs
    - D) Replacing version control

---
## 11) Quick Reference

- Create: `terraform workspace new <name>`
- Switch: `terraform workspace select <name>`
- List: `terraform workspace list`
- Show: `terraform workspace show`
- Delete: `terraform workspace delete <name>`
- Use variables: `terraform apply -var-file=env/<ws>.tfvars`
- Access name in code: `terraform.workspace`

---
## 12) Summary

- Workspaces provide **state isolation** for multiple deployments from the **same configuration**.
- Pair workspaces with **tfvars**, **remote backends**, and **CI** to create clean, repeatable environment management.
- For **strong isolation**, use separate accounts/projects and backends; treat workspaces as a convenient partitioning of state, not a security boundary.

