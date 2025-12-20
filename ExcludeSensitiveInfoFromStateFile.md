
}


output "db_password_out" {
  value     = var.db_password
  sensitive = true
}
```


**Why it doesn’t work:**  
- Output and logs show `(sensitive value)`, but **state still contains** the cleartext or serialized value.  
- Docs: the `sensitive` argument **redacts** values in CLI/HCP UI; it’s **not** a state protection mechanism.[^hashi-manage]


---


## 4) ❌ **Does NOT prevent secrets in state: Environment variables (`TF_VAR_*`)**


**What it is:** Supply secrets via shell env (e.g., `TF_VAR_db_password`). Terraform resolves them to variables.[^tf-env]


### Example:
```bash
export TF_VAR_db_password='[Credentials]'
terraform apply
```


**Why it doesn’t work:**  
- It keeps secrets out of Git/config files, but **once Terraform assigns the value**, it usually becomes part of **resource arguments** and thus **state**.  
- This method addresses **injection**, not **state storage**.[^hashi-manage]


---


## 5) ❌ **Does NOT prevent secrets in state: SOPS‑encrypted `.tfvars` files**


**What it is:** Encrypt `.tfvars.json` with SOPS and load it using the **SOPS provider**; Terraform decrypts at runtime.[^sops-reg]


### Example:
```hcl
provider "sops" {}


data "sops_file" "tfvars" {
  source_file = "secrets.auto.tfvars.json.enc"
}


locals {
  secrets = jsondecode(data.sops_file.tfvars.raw)
}


resource "my_db_resource" "db" {
  admin_password = local.secrets.db_password
}
```


**Why it doesn’t work (alone):**  
- Decrypting at runtime is great for Git, but **the decrypted value** typically flows into resource attributes and ends up in **state**.  
- SOPS provider docs explicitly warn to use **secure remote state**; encryption of the input file **does not** stop state from containing the value.[^sops-reg]


> To truly keep secrets **out of state**, combine SOPS with **ephemeral variables** or **write‑only** attributes.


---


## 6) ❌ **Does NOT prevent secrets in state: Vault / Cloud Secret Managers (by themselves)**


**What it is:** Retrieve secrets at apply‑time from Vault, AWS Secrets Manager, Azure Key Vault, Google Secret Manager.[^vault-reg]


### Example (AWS Secrets Manager):
```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db-password"
}


resource "aws_db_instance" "main" {
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```


**Why it doesn’t work (alone):**  
- When the secret is **read into Terraform** and assigned to a resource attribute, it’s typically **stored in state**.  
- Vault provider docs: interacting with Vault can cause secrets to be **persisted in Terraform state**; treat state as sensitive.[^vault-reg]


> Use secret managers **together with** `ephemeral` or write‑only to keep secrets out of state.


---


## 7) **State storage hygiene (still essential)**


Even with epoxy‑tight patterns above, keep state secure:


- Prefer **remote encrypted state** (e.g., S3 + KMS + DynamoDB locking; HCP Terraform encrypts state at rest and uses TLS in transit).[^hashi-manage]  
- Lock down IAM and enable audit logging; **never commit** `terraform.tfstate`.[^hashi-manage]


---
## 9) Quick reference: **What protects state, what doesn’t**

| Method | Prevents storage in state? | Notes |
|---|---|---|
| `ephemeral = true` | **Yes** ✅ | Omitted from state & plan; best option.[^hashi-manage] |
| Write‑only (`value_wo`) | **Yes** ✅ | Never in state/plan; provider/resource support required.[^tfe-variable] |
| `sensitive = true` | **No** ❌ | Only redacts CLI/UI; state still contains value.[^hashi-manage] |
| Environment (`TF_VAR_*`) | **No** ❌ | Keeps secrets out of code; not out of state.[^tf-env] |
| SOPS‑encrypted files | **No** ❌ | Protects at rest in Git; decrypted values usually enter state.[^sops-reg] |
| Vault / Secrets Manager | **No** ❌ (alone) | Reading into Terraform often persists to state; combine with ephemeral/write‑only.[^vault-reg] |

---

## 10) TL;DR

- **Use `ephemeral` variables** or **write‑only arguments** wherever possible. That’s the only way to **guarantee** secrets won’t be in state.[^hashi-manage][^tfe-variable]  
- Everything else (sensitive, env vars, SOPS, secret managers) solves **where secrets come from**, not **where Terraform stores them**.  
- Still secure your state: **remote + encryption + IAM**.[^hashi-manage]
