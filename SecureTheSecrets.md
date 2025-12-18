# Secure Ways to Store and Access Secrets in Terraform

Managing sensitive information (API keys, credentials, tokens) securely is critical in Terraform workflows. Below are the recommended approaches ranked from **best to worst**, with pros and cons.

---

## 1. **Use a Secret Manager (e.g., HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)**
**Example:**
```hcl
provider "vault" {}
data "vault_generic_secret" "example" {
  path = "secret/data/myapp"
}
```
**Pros:**
- Centralized, secure, audited storage
- Dynamic secrets and automatic rotation
- Fine-grained access control
**Cons:**
- Requires additional infrastructure and setup
- Adds complexity to CI/CD pipelines

---

## 2. **Terraform Cloud / HCP Terraform Workspace Variables**
**Example:**
- Store sensitive variables in Terraform Cloud workspace as **sensitive**.
- Access automatically during runs.
**Pros:**
- Built-in encryption and RBAC
- No secrets in code or VCS
- Easy integration with Terraform workflows
**Cons:**
- Requires Terraform Cloud or Enterprise subscription

---

## 3. **Environment Variables**
**Example:**
```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="secret"
terraform apply
```
**Pros:**
- Simple and widely supported
- Works well in CI/CD pipelines
**Cons:**
- Can leak via process lists or logs
- Requires secure handling in pipeline configs

---

## 4. **Encrypted Files (e.g., SOPS, GPG-encrypted tfvars)**
**Example:**
- Store secrets in `secrets.enc.json` and decrypt at runtime.
**Pros:**
- Version-controlled but encrypted
- Can integrate with KMS for key management
**Cons:**
- Adds complexity for decryption in CI/CD
- Risk if encryption keys are mishandled

---

## 5. **Backend Config Files (with .gitignore)**
**Example:**
```hcl
# backend.hcl
bucket = "mybucket"
key    = "state.tfstate"
region = "us-east-1"
```
Run:
```bash
terraform init -backend-config=backend.hcl
```
**Pros:**
- Keeps secrets out of main code
- Easy to manage per environment
**Cons:**
- Must ensure files are excluded from VCS
- Still plaintext unless encrypted

---

## 6. **Inline Hardcoding in Code (Worst Practice)**
**Example:**
```hcl
provider "aws" {
  access_key = "AKIA..."
  secret_key = "secret"
}
```
**Pros:**
- None (except simplicity)
**Cons:**
- Secrets exposed in VCS
- Violates security best practices

---

# ✅ Summary Table
| Rank | Method                                | Pros                                   | Cons                                      |
|------|---------------------------------------|----------------------------------------|-------------------------------------------|
| 1    | Secret Manager (Vault, AWS SM)       | Secure, audited, dynamic secrets      | Requires setup and infra                 |
| 2    | Terraform Cloud Sensitive Variables  | Built-in encryption, RBAC             | Needs Terraform Cloud subscription       |
| 3    | Environment Variables                | Simple, CI/CD friendly                | Risk of leaks in logs/process lists      |
| 4    | Encrypted Files (SOPS/GPG)           | Version-controlled, encrypted         | Complexity in decryption                 |
| 5    | Backend Config Files (.gitignore)    | Keeps secrets out of main code        | Plaintext unless encrypted               |
| 6    | Inline Hardcoding                    | None                                  | Secrets exposed in VCS                   |

---

# ✅ Best Approach (Descending Order)
1. Secret Manager (Vault, AWS Secrets Manager, Azure Key Vault)
2. Terraform Cloud Sensitive Variables
3. Environment Variables
4. Encrypted Files (SOPS/GPG)
5. Backend Config Files
6. Inline Hardcoding (Avoid!)


---

## Appendix: Consul Backend `-backend-config` Example
When using the **Consul** backend, you can supply required arguments at init time with multiple `-backend-config` flags:

```bash
terraform init   -backend-config="address=demo.consul.io"   -backend-config="scheme=https"   -backend-config="path=example_app/terraform_state"
```

**Notes:**
- You can pass **multiple** `-backend-config` flags; later flags override earlier ones.
- Keep secrets (e.g., Consul tokens) **out of code**. Provide them via environment variables or a credential helper, not inline.
- This pattern leverages Terraform’s **partial backend configuration**, allowing you to omit sensitive arguments from the `backend` block and provide them securely at init time.


---

# How to Use Stored Values in Terraform (Code Examples)
Below are concise code patterns showing **how your Terraform config reads/uses secrets** for each method.

## 1) Secret Manager — HashiCorp Vault
Fetch a secret and use it (e.g., DB password). *Note:* Reading secrets via the Vault provider persists values in plan/state; protect state and limit exposure.  
**Docs:** Vault provider & best practices.  

```hcl
# Providers (Vault, and the cloud you target)
provider "vault" {
  address = var.vault_addr  # set via env VAULT_ADDR or variable
}

# Read a KV secret (KV v2 path example)
data "vault_generic_secret" "db" {
  path = "secret/data/prod/db"
}

# Use the secret value
variable "db_username" { type = string }

resource "aws_db_instance" "prod" {
  # ... other config
  username = var.db_username
  password = data.vault_generic_secret.db.data["password"]
}
```

> Reference: HashiCorp Vault provider docs and guidance about state exposure.  

## 2) Secret Manager — AWS Secrets Manager
Retrieve the current secret version and use it (e.g., `DB_PASSWORD`).

```hcl
# Get secret metadata (optional)
data "aws_secretsmanager_secret" "db" {
  arn = var.db_secret_arn
}

# Retrieve latest secret value
data "aws_secretsmanager_secret_version" "db_current" {
  secret_id = data.aws_secretsmanager_secret.db.id
}

# Use the secret value (JSON string -> map)
locals {
  db_secret_map = jsondecode(data.aws_secretsmanager_secret_version.db_current.secret_string)
}

resource "aws_db_instance" "prod" {
  # ... other config
  username = local.db_secret_map["username"]
  password = local.db_secret_map["password"]
}
```

## 3) Secret Manager — Azure Key Vault
Fetch a secret at runtime and use its value. *Note:* the value is stored in state; mark outputs sensitive and secure state storage.

```hcl
# Read existing Key Vault and secret
data "azurerm_key_vault" "kv" {
  name                = var.key_vault_name
  resource_group_name = var.key_vault_rg
}

data "azurerm_key_vault_secret" "db_password" {
  name         = var.secret_name
  key_vault_id = data.azurerm_key_vault.kv.id
}

resource "azurerm_postgresql_flexible_server" "prod" {
  # ... other config
  administrator_login          = var.db_user
  administrator_login_password = data.azurerm_key_vault_secret.db_password.value
}
```

## 4) Terraform Cloud / HCP Terraform Workspace Variables
Declare variables in code; set **sensitive values** in the workspace (UI/API/tfe provider). CLI authenticates via `terraform login` or `TF_TOKEN_*` and does **not** expose secrets in code.

```hcl
# variables.tf
variable "db_password" { type = string, sensitive = true }

# main.tf
resource "aws_db_instance" "prod" {
  # ... other config
  username = var.db_user
  password = var.db_password  # value set in HCP Terraform workspace vars
}

# Optional: manage workspace variables via tfe provider
provider "tfe" {}
resource "tfe_variable" "db_pass" {
  workspace_id = var.tfc_workspace_id
  key          = "db_password"
  sensitive    = true
  category     = "terraform"
  value_wo     = var.db_password  # write-only; not stored in state
}
```

## 5) Environment Variables
Use `TF_VAR_*` for input variables and provider-native envs for credentials. Keep them out of code.

```bash
# Shell (set before running Terraform)
export TF_VAR_db_password="s3cr3t"
export AWS_PROFILE=prod
# or explicit creds if needed:
export AWS_ACCESS_KEY_ID=AKIA...; export AWS_SECRET_ACCESS_KEY=...
```

```hcl
# variables.tf
variable "db_password" { type = string, sensitive = true }

# main.tf
resource "aws_db_instance" "prod" {
  username = var.db_user
  password = var.db_password   # comes from TF_VAR_db_password
}

# provider picks credentials from env/profile
provider "aws" {
  region = var.aws_region
}
```

## 6) Encrypted Files (e.g., SOPS `.tfvars.json` / YAML)
Decrypt at run time and feed values into Terraform.

```hcl
# Using sops provider to decrypt an encrypted file
# (alternatively: decrypt to a temp .auto.tfvars.json in CI)

data "sops_file" "env" {
  source_file = "secrets/prod-enc.yaml"  # encrypted
}

locals {
  secrets = yamldecode(data.sops_file.env.raw)
}

resource "aws_memorydb_user" "lambda" {
  user_name = "lambda-user-prod"
  authentication_mode {
    type      = "password"
    passwords = [local.secrets["MEMORY_DB_PASSWORD"]]
  }
}
```

## 7) Backend Config Files / Flags (for state only)
Provide backend settings at init, and keep tokens in env (e.g., Consul). Not consumed by resources; this controls **where state lives**.

```hcl
# backend block without secrets
terraform {
  backend "consul" {}
}
```

```bash
# Supply backend params and token via env
export CONSUL_HTTP_TOKEN=...   # token
terraform init   -backend-config="address=demo.consul.io"   -backend-config="scheme=https"   -backend-config="path=example_app/terraform_state"
```

## 8) Inline Hardcoding (avoid)
```hcl
provider "aws" {
  access_key = "AKIA..."   # ❌ do not hardcode
  secret_key = "secret"     # ❌ do not hardcode
}
```

---

## Notes & gotchas
- **Backends** are evaluated before variables; avoid `var.*` in backend blocks and use `-backend-config` instead.
- **State exposure:** Many data sources (Vault, Key Vault, Secrets Manager) will place retrieved values in state/plan. Secure state (remote, encrypted, least-privileged access) and mark outputs `sensitive`.
- **Prefer identity-based auth:** IRSA (AWS), Managed Identity (Azure), Workload Identity (HCP Terraform) for short-lived credentials.

