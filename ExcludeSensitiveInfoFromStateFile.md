
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
